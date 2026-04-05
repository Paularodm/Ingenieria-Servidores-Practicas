# Memoria de Prácticas: Ingeniería de Servidores
**Autor:** Paula Rodriguez Montoro
**Curso:** 2025/2026
**Repositorio:** [Enlace a mi GitHub](https://github.com/tu-usuario/tu-repo)

---

## BLOQUE 1: Configuración del Entorno y Administración

### Práctica 2: Configuración de LVM y RAID
Se desea instalar un servicio de gestión documental en el servidor. Se espera que este servicio
precise de una cantidad espacio de almacenamiento creciente con el tiempo, pudiendo llegar a
ser considerable. Por otro lado, el contenido será crítico, por lo que se desea proporcionar
algún mecanismo de respaldo ante fallos en el dispositivo de almacenamiento.

Para ello utilizaremos las herramientas de RAID y LVM. En particular, raid 1, para agrupar dos discos físicos en un único dispositivo virtual, de manera que si uno de ellos se avería o falla el sistema continue operando sin pérdida de información gracias a la replica exacta en el segundo disco. Y sobre esta base segura del RAID, implementaremos LVM para gestionae el espacio de forma flexible, mediante physical volumes, volume groups y logical volumes. Además podremos expadir la capacidad de almacenamiento "en caliente" sin interrumpir el servicio. 

#### Pasos realizados y Capturas de pantalla:

#### RAID: 

1. Añadir los discos en Virtual Box.
2. Comprobar que se encuentran disponibles con lsblk.
   
![Primer lsblk para comprobar que se han creado los discos](img/P2_primer_lsblk.png)

3. Crear el raid utilizando el comando mdadm.

![Creamos el raid utilizando el comando mdadm.](img/P2_creacionraid_mdadm.png)

* `mdadm`: Herramienta para gestionar RAID en Linux.
* `--create /dev/md0` : Define el nombre del nuevo dispositivo RAID en el sistema.
* `--level=1` : Selecciona el nivel RAID 1, esencial para datos críticos ya que si un disco falla, la información permanece intacta en el otro.
* `--raid-devices=2` : Indica el número de discos físicos que componen el array.
* `/dev/sdb /dev/sdc` : Son los "ingredientes" o dispositivos de bloque que hemos añadido previamente.
  
4. Comprobar el raid se ha creado correctamente.

![alt text](img/P2_estado_del_raid_tras_crearlo_cat.png)

Podemos observar que el raid se ha creado correctamemte pues:
* `md0 : active raid1 sdc[1] sdb[0]` : Confirma que estan activos y participando en "espejo".
* `[UU]` :  Ambos discos estan operativos: U (Up)


![alt text](img/P2_estado_raid1_mdadm_detail.png)

Tras la ejecución de `mdadm`, se ha verificado mediante `mdadm --detail` que el estado del array es `active` y que cuenta con dos unidades sincronizadas `active sync`. De esta manera, estamos cumpliendo con el requisito de proporcionar un mecanismo de respaldo ante fallos físicos en el almacenamiento.

5. Persistencia en la configuración. 

Para realizar la configuración del RAID y la persistencia en `/etc/mdadm.conf`, se han empleado privilegios de superusuario mediante el comando `sudo`, garantizando así el acceso a los archivos de configuración del sistema y a los dispositivos de bloque.

![alt text](img/image.png)

#### LVM:

Una vez hemos configurado la base segura mediante RAID 1, que nos garantiza la réplica exacta de los datos y la tolerancia a fallos físicos, vamos a implementar LVM para dotar al sistema de la flexibilidad necesaria ante un crecimiento considerable del volumen de información.

Como punto de montaje definitivo se ha seleccionado el directorio /var ya que según el estándar de jerarquía de ficheros /var es el destino de datos variables y dinámicos, lo que encaja con el perfil de un sistema de gestión documental. Además, al utilizar LVM, podremos expandir el espacio de /var "en caliente" si el volumen de documentos crece más de lo previsto y aislar los datos del servicio en su propio volumen lógico. Evitamos así que un crecimiento excesivo de archivos bloquee la partición raíz (/), garantizando la máxima disponibilidad del servidor.

6. Crear el physical volume (pv).

Este paso prepara el dispositivo RAID para ser integrado en LVM.

![pvcreate](img/P2_pvcreate.png)

Para ver si se ha creado correctamente: `sudo pvdisplay`

![pvdisplay](img/P2_pvdisplay.png)

Podemos ver que se ha creado el pv, pero que no tiene asociado ningún vg: `Allocatable=NO`


7. Crear el volume group (vg).

Usaremos este componente para agrupar el espacio de almacenamiento del RAID, para gestionar de forma centralizada los logical volumes (lv) que se dispongan sobre él.

Es una buena práctica separar pv de distintas características en vg distintos. En nuestro caso, no mezclaremos en un mismo vg dispositivos que emplean RAID y dispositivos que no. Si hacemos esto cuando se escriba en cierto lv, como no se sabe en qué pv está y entonces no sabremos si se ha empleado el RAID o no.

![vgcreate and vgdisplay](img/P2_vgcreate_and_vgdisplay.png)

Si volvemos a ejecutar de nuevo `pvdisplay` vemos como ahora si nos aparece como `VG Name` `vg_datos` y `Allocatable=yes`

![pvdisplay](img/P2_pvdisplay_con_vg_creado.png)

8. Crear el logical volume (lv).

Ahora que tenemos el vg que solo contiene espacio seguro del RAID, asignamos el espacio que usaremos realmente (lv): 
   
![lvcreate](img/P2_lvcreate.png)
   
* `-L 5G` : indica el tamaño del lv
* `-n lv_documentos` : nombre del lv
* `vg_datos` : situado en el vg vg_datos
  
Veamos que esta creado con las propiedades indicadas:

![lvdisplay](img/P2_lvdisplay.png)

9. Proporcionar sistema de archivos.

Hasta ahora, tenemos el "contenedor" de almacenamiento, pero el sistema operativo necesita una estructura organizada para poder leer y escribir los archivos de tu servicio de gestión documental.

![mkfs.ext4](img/P2_mkfs.png)

10. Preparación del Punto de Montaje Temporal

Montamos el lv en `/mnt` para tener un lugar donde volcar los datos de `/var` antes de la sustitución oficial.

![Nuevo mount sobre /mnt](img/P2_nuevo_mount.png)
![lsblk](img/P2_lsblk_montado_en_mnt.png)

11. Cambio al Modo de Mantenimiento
    
Necesitamos dar este paso ya que `/var` contiene bases de datos y logs que se escriben cada segundo. Si copiamos `/var` mientras el sistema está en modo multiusuario, la copia podría ser inconsistente.

Comando: `sudo systemctl isolate runlevel1.target`

12. Sincronización de Datos

Ahora si, copiamos el contenido de `/var` en `/mnt` asegurándonos de mantener los metadatos.

![Contendio de var copiado](img/P2_copia_contenido_var_mnt.png)

Podemos ver que ambos directorios tienen el mismo numero de archivos, lo que es un indicador de que hubo exito. 

![comparacion var y mnt](img/P2_comprobacion_del_n_arch_en_var_mnt.png)

De todas formas, comprobamos con df -h: 

![df -h](img/P2_df-h_tras_copia_var.png)

13. Sustitución del Directorio /var
    
Ahora que los datos están seguros en el LVM sobre RAID 1, podemos "mover" el `/var` original para que el sistema use uno nuevo. 
Renombramos el original para no borrarlo y creamos el nuevo punto de entrada.

![alt text](img/P2_cambio_var_a_varold_y_creo_nuevo_var.png)

14. Montaje Definitivo
    
Desvinculamos el RAID de la carpeta temporal y lo anclamos a su destino final. 

![alt text](img/P2_desmonto_en_mnt.png)

![alt text](img/P2_monto_en_var.png)

![alt text](img/P2_df-h_definitivo_var.png)

15. Persistencia en la configuración.
    
Editamos el archivo `/etc/fstab` para que el servicio no falle tras reiniciar el sistema, es decir, se mantega el montaje.

![/etc/fstab](img/P2_etc_fstab_var.png)


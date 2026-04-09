# Memoria de Prácticas: Ingeniería de Servidores
**Autor:** Paula Rodriguez Montoro
**Curso:** 2025/2026
**Repositorio:** [Enlace a mi GitHub](https://github.com/Paularodm/Ingenieria-Servidores-Practicas)

---

## BLOQUE 1: Configuración del Entorno y Administración

### Práctica 3: Acceso seguro al servidor: Firewall + SSHD

#### 4.2.1.- Ejercicio. Configuración de SSHD: 
#### Pasos realizados:

1. Modificación del puerto en el que se ejecuta SSHD:
   
   1.1. Elegimos un puerto que no se este usando, por ejemplo yo he cogido el 2026.<br>
   Comprobamos que no se este usando en el archivo `/etc/services`.<br>
   1.2. Actualizamos el firewall para que permita el acceso a dicho puerto: `firewall-cmd --permanent --add-port=<puerto>/tcp`. <br>
   1.3. Cambiamos el puerto de acceso sshd en el archivo `/etc/ssh/sshd_config` . Indicamos que tiene acceso seguro con `semanage` y actualizamos con `sudo systemctl restart`. <br>

2. Conceder acceso remoto por llave pública al usuario paula.
   
   1.1. Generación del par de llaves en el Host .<br>
   El primer paso es crear nuestra identidad digital basada en criptografía asimétrica.
   Ejecutamos en la terminal del anfitrión el comando `ssh-keygen -t rsa -b 4096`.<br>
   1.2. Transferencia de la llave pública al servidor.<br>
   1.3. Validación mediante ejecución remota. <br>
   
    Capturas:

![poershellssh](img/P3_powershell_ssh.png)
> *Host: Ssh efectivo y resultado de ejecucion ls -la ~.*

![ssh_desdeMV](img/P3_ssh_desdeMV.png)
> *MV: Resultado de ejecucion ls -la ~.*
---
# 1.0 El proyecto.

Voy a desplegar una máquina con Ansible en AWS, que me permita controlar de forma remota.

## 1.1 Instancia EC2 de Ansible.

- Nombre: `Ansible_control_EC2`.
- AMI: `Ubuntu Server 22.04 LTS (HVM), SSD Volume Type`.
- Tipo de instancia: `t2.micro`.
- Crear un nuevo par de claves (RSA, .pem).

**Editamos la configuración de red:**
  ![image](https://github.com/user-attachments/assets/8884a75c-1307-4378-9c9c-802c6c5e192b)

y ya. La creamos.

## 1.2 Instancias (3) EC2 de CentOs.
- Número de instancias: `3`.
- Nombre: `servidores_de_ansible`.
- AMI: `CentOS Stream 9 (x86_64)`.
- Tipo de instancia: `t2.micro`.
- Par de clave: `cliente`,`RSA`,`.pem`.

Editamos igualmente la configuración de red:

![image](https://github.com/user-attachments/assets/ffe8a2c7-1c4d-4255-9669-8ae45d41285b)

La primera regla es que desde mi IP pueda hacer ssh para conectarme a la instancia y la otra que el pueda conectarse desde la máquina de Ansible (he puesto Ansible_SG)

Una vez creadas, **les voy a cambiar el nombre manualmente a cada máquina para que se distingan**:

![image](https://github.com/user-attachments/assets/1b64d87e-58e8-4434-8323-d6f96e32bcff)

## 1.3 Instalar Ansible en la máquina.

Estas máquinas ya tienen SSH, todas. **Voy a acceder a Ansible_Control_EC2**

Es importante recalcar, que cada vez que hagamos SSH a nuestra instancia EC2, nos preguntará:

`Are you sure you want to continue conecting (yes/no[fingerprint])?`

Esto significa que básicamente nos muestra el ID de la máquina, SHA256: xxxxxxxxx.

Cuando yo vuelva a hacer SSH no nos lo volverá a pedir, eso, es porque hay un fichero que almacena esas fingerprints.

![image](https://github.com/user-attachments/assets/b59262e0-b28c-492e-80a1-84532f11fdd9)

Voy a mostrar el fichero para que lo pueda leer:

![image](https://github.com/user-attachments/assets/a07259e8-9360-4f45-9ae2-11ba3e144156)

Voy a eliminar el contenido del fichero.

```
cat /dev/null > ~/.ssh/known_hosts
```

Entonces, cuando yo vuelva a:

```
ssh -i "ansible.pem" ubuntu@ec2-34-228-37-255.compute-1.amazonaws.com
```

> [!TIP]
>Sin embargo, el fingerprint que almacene dependerá de la máquina de AWS y de su configuración en ese momento. Si la máquina no ha cambiado su configuración o clave SSH (por ejemplo, no se ha renovado la clave del servidor de la instancia de AWS), el fingerprint debería ser el mismo que el anterior. Si la instancia de AWS se ha reiniciado o ha sido reemplazada, o si AWS ha cambiado la clave SSH, el fingerprint podría ser distinto.
>
>En resumen:
>
>- Si la clave SSH de la máquina de AWS no ha cambiado, el fingerprint será el mismo.
>- Si la clave ha cambiado (por ejemplo, si la máquina de AWS fue reemplazada o se ha cambiado la clave SSH), el fingerprint será diferente.
>

**Ansible hace pues un proceso similar para conectarse a las máquinas.**

En cualquier caso, ya estaremos dentro de la máquina de Ansible, así que vamos a buscar Ansible download.

Tenemos la opción de hacerlo usando Python:
https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#pipx-install

En cualquier caso:

>[!IMPORTANT]
>Este es el importante:
>https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html
>
>Y los comandos para Ubuntu son estos:
>```
> sudo apt update && sudo apt install -y software-properties-common && sudo add-apt-repository --yes --update ppa:ansible/ansible && sudo apt install -y ansible
>```

Básicamente, es como una librería de Python, usa el 2 y el 3.

Voy a ver que se haya descargado:

```
ansible --version
```

## 1.4 Crear Inventario. (Y cosas relacionadas con el inventario).
>[!TIP]
>Recordemos, que el inventario, no es más que un fichero en el que se almacenan los Hosts a los que se les va a aplicar los comandos o lo que sea.
>
>Y también recordemos que el inventario puede estar en 3 formatos diferentes:
>  - Inventario Dinámico.
>  - **YAML o YML. (la que vamos a utilizar)**
>  - Texto plano, .ini.

Entonces, ahora voy a crear un directorio en el que voy a poner el inventario.
```
mkdir prueba_ansible
```

https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html

Es importante saber, que hay variables:
- ansible_host.
- ansible_port.
- ansible_user.
- ansible_password. (ESTO NO SE UTILIZA NUNCA, JAMÁS).
- ansible_ssh_private_key_file. (Lo que SI, vamos a utilizar).

### 1.4.1 Problema con el que me he topado, pérdida de las claves .pem .

En resumen, formatee el ordenador y me he quedado sin las claves. No hay forma de cambiar el par-clave de la instancia, tampoco podemos contactar con Amazon en caso de que se nos pierda. Es en teoría imposible.

Yo aquí tengo varias opciones:
- Eliminar las instancias y crearlas de nuevo.
- Acceder AL VOLUMEN de la máquina y editar un fichero.

>[!IMPORTANT]
>Entonces, en este volumen, habrá un fichero: `.shh/authorized_keys`.
>Y para resumirlo mucho, en efecto, tenemos que acceder al volumen ¿Cómo?
>
>1. Tenemos que apagar la máquina afectada, la que no tenemos el par-clave.
>2. Hacer otra instancia, de rescate.
>3. Desacoplar el volumen de la máquina afectada, y conectarlo a la instancia de rescate.
>4. Editar el fichero en cuestión para añadirle el par de clave.
>5. Una vez editado, acoplamos de nuevo el volumen con el fichero editado con la instancia afectada.

**Para colmo, he terminado la instancia sin querer** Ya que me he puesto con este reto. Voy a hacerlo, terminarlo y demostrar que se puede.

**Es importante, que los dos estén en la misma zona de disponibilidad**.
Es decir, que si mi máquina está en `Zona de disponibilidad
us-east-1b` y mi otra máquina en otro, no funcionará.

Entonces, creo la máquina de rescate, sin más. Voy a también darle nombre a los volúmenes:

**PASO 1:**

![image](https://github.com/user-attachments/assets/856a1c80-3b74-4070-b454-1cfbe5be4e3f)

**********************************

**PASO 2:**

![image](https://github.com/user-attachments/assets/9206944c-5171-468d-86fd-556a8400878f)

Cómo hemos dicho la máquina afectada tiene que estar apagada, y desasociamos ese volumen.

Nos vamos a volúmenes:

**PASO 3**

![image](https://github.com/user-attachments/assets/090c5933-0b78-410b-8e48-2ac717362c3b)

![image](https://github.com/user-attachments/assets/a5ddaffe-2800-4bdd-85e3-2d70c59496fd)

Una vez lo desasocie, voy a asociarlo, pero con la máquina de rescate.

**PASO 4**

![image](https://github.com/user-attachments/assets/57dc26e7-052a-412a-a913-fb671276a2a7)

![image](https://github.com/user-attachments/assets/26fcc287-c5d3-4208-85fe-70295aadae43)

En nombre de dispositivo:

>[!TIP]
>Esto es un concepto interesante a explicar, básicamente en Windows las unidades de almacenamiento o particiones, usan nombres como `C: `, `E: `, `D: `.
>
>Es básicamente la nomenclatura de las particiones, volumenes, etc...
>En linux, no se usa esa "nomenclatura", usa otra:
>
> - `/dev/sda`, `/dev/sdb`, etc.: Representan discos físicos
>   - `/dev/sda1`, `/dev/sda2`, etc.: Son particiones dentro de un disco.
>
> Entonces, en nombre de dispositivo te dice que:
> - `/dev/sda1` ya está siendo utilizado, bueno, pues elijo el `/dev/sdd` y no pasa nada.
> - Es más, si tu pruebas con el /dev/sdb, no te va a funcionar aunque no aparezca como que no está disponible.

Ahora ya tenemos dos disco duros/volumenes a una misma instancia. Vamos a conectarnos...
Voy a abrir la terminal de Gitbash desde el escritorio, que es donde tengo la clave .pem de la máquina de rescate.

https://codigoencasa.com/aws-recuperar-llave-pem-curso-aws/

Ya estoy dentro de la máquina y ahora voy a hacer una carpeta de montaje de este nuevo volumen.
Hemos conectado a la máquina, pero no está montado.

`sudo su`

```
mkdir /mnt/tmp
```

y este comando lo he modificado A MIS preferencias, porque anteriormente en "nombre de dispostivo, pusimos `/dev/sdd`".

Si utilizo estos dos comandos, no va a funcionar: `mount /dev/sdd1 /mnt/tmp` o `mount /dev/sdd /mnt/tmp`

voy a poner este comando, para `lsblk`

Output:
```
NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0      7:0    0 26.3M  1 loop /snap/amazon-ssm-agent/9881
loop1      7:1    0 73.9M  1 loop /snap/core22/1722
loop2      7:2    0 44.4M  1 loop /snap/snapd/23545
xvda     202:0    0    8G  0 disk
├─xvda1  202:1    0    7G  0 part /
├─xvda14 202:14   0    4M  0 part
├─xvda15 202:15   0  106M  0 part /boot/efi
└─xvda16 259:0    0  913M  0 part /boot
xvdd     202:48   0    8G  0 disk
├─xvdd1  202:49   0    7G  0 part
├─xvdd14 202:62   0    4M  0 part
├─xvdd15 202:63   0  106M  0 part
└─xvdd16 259:1    0  913M  0 part
```

Entonces, el volumen está identificado como xvdd y su partición principal es xvdd1.
Esto se debe a que, aunque en AWS haya puesto nombre de dispositivo `sdd`, como son versiones de Ubuntu más recientes pues se utiliza esta otra nomenclatura.

Literalmente lo que pone en el AWS:

![image](https://github.com/user-attachments/assets/4ee67dc4-da76-492b-8dd6-cc11d435f64a)

Finalmente se usa este comando:

```
mount /dev/xvdd1 /mnt/tmp
```

>[!TIP]
>¿Porqué se usa `tmp` en vez de `temp`?
>Ni idea. ChatGPT lo usa porque sí.
>

Una vez montado, voy a copiar el fichero `./.ssh/authorized_keys` DEL VOLUMEN DE RESCATE, hacia el afectado.

```
cp ./.ssh/authorized_keys /mnt/tmp/home/ubuntu/.ssh/authorized_keys
```

Una vez hecho eso, voy a cerrar la sesión, `exit` y `exit`.

![image](https://github.com/user-attachments/assets/5acf74d5-0590-4441-aba7-7c41719a9779)

Apago la máquina, me vuelvo a volúmenes y voy a desaociar el volumen ese y lo voy a volver a asociar a su instancia correspondiente.

**PASO 5**

![image](https://github.com/user-attachments/assets/8e9e7c91-205d-42f7-b52a-66d3a3b0d7c5)

**PASO 6**
![image](https://github.com/user-attachments/assets/3e95acb3-963e-439e-a635-43322a1b5f8b)

Pero esta vez, muy importante, recordemos que "nombre de dispositivo" antes tenía reservado el `/dev/sda1`, estaba ocupado porque allí estaba antes su volumen, ahora, evidentemente, no tiene nada, ningún volumen, pues lo pondremos allí.


![image](https://github.com/user-attachments/assets/a3b7bb55-0acc-4613-955d-c2d241c80c41)

eso es todo, ahora solo tenemos que iniciar la instancia y comprobar que efectivamente ha funcionado.

Si me voy a conectar...

El comando está compuesto por `ssh -i "el nombre anterior del archivo clave"`

Eso NO es lo que queremos, nosotros queremos usar el .pem nuevo, de la otra máquina de rescate, pues ya está, solo modificamos el comando anterior y ya:

`ssh -i "rescate.pem" ubuntu@ec2-44-223-xxx-xxx.compute-1.amazonaws.com`.

Como podemos comprobar hemos podido acceder efectivamente a la máquina:

![image](https://github.com/user-attachments/assets/69c54f3c-d37c-43f7-aea1-ae4deadc2740)

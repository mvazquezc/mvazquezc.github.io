# Introducción a la seguridad en contenedores

---

## Mario Vázquez

<!--<img src="images/profile_picture.jpg" style="float: right"/> -->

### Solutions Engineer
### Red Hat - Telco Engineering

<!-- .slide: style="text-align: left;"> -->
<i class="fa-solid fa-globe"></i><a href="https://linuxera.org">  linuxera.org</a><br>
<i class="fa-solid fa-envelope"></i>  mario@redhat.com<br>
<i class="fa-brands fa-twitter"></i><a href="https://twitter.com/mvazce">  @mvazce</a><br>
<i class="fa-brands fa-github"></i><a href="https://github.com/mvazquezc">  @mvazquezc</a><br>
<i class="fa-brands fa-linkedin-in"></i><a href="https://www.linkedin.com/in/mariovazquezcebrian/">  @mariovazquezcebrian</a>

---

## Objetivo de la presentación

<!-- .slide: style="text-align: left; font-size: 18px;"> -->
- Conocer qué son y para qué se utilizan las _capabilities_ de Linux. 
- &shy;<!-- .element: class="fragment" data-fragment-index="1" --> Conocer cómo se utilizan las _capabilities_ en los contenedores. 
- &shy;<!-- .element: class="fragment" data-fragment-index="2" --> Conocer qué son y para qué se utilizan los _Secure Compute Profiles (seccomp)_.
- &shy;<!-- .element: class="fragment" data-fragment-index="3" --> Conocer cómo podemos crear nuestros propios perfiles _seccomp_.
- &shy;<!-- .element: class="fragment" data-fragment-index="4" --> Conocer cómo podemos utilizar a nuestro favor las _capabilities_ y los perfiles _seccomp_ en entornos Kubernetes. 

---

# Linux _Capabilities_

---

## Linux _Capabilities_
<!-- .slide: style="text-align: left; font-size: 18px;"> -->

- A la hora de realizar comprobaciones de permisos, las implementaciones de UNIX tradicionales distinguían entre dos tipos de procesos:

  - Procesos Privilegiados: Su _effective user ID_ es **0**, conocido también como **superuser** o **root**.
  - Procesos No-Privilegiados: Su _effective user ID_ no es **0**.

- Los procesos privilegiados se saltan todas las comprobaciones de permisos del Kernel.
- Los procesos no privilegiados están sujetos a una comprobación de todos sus permisos basándose en los credenciales del proceso, habitualmente, su _effective UID_, _effective GID_ y grupos suplementarios.
- En el Kernel 2.2, Linux dividió los privilegios tradicionalmente asociados al **superusuario** en distinas unidades, conocidas como _capabilities_, las cuales pueden ser activadas o desactivadas de manera independientemente.
- Las _capabilities_ son un atributo de cada hilo. <img src="images/caps.png" height="100px" width="300px" style="float: right"/> 

---

## Linux _Capabilities_
<!-- .slide: style="text-align: left; font-size: 18px;"> -->

- Ejemplos de capabilities:

  - **_NET_RAW_**: Permite usar _sockets_ _RAW_ y _PACKET_.
  - **_CHOWN_**: Permite hacer cambios arbitrarios a _UIDs_ y _GIDs_ de ficheros.
  - **_NET_ADMIN_**: Permite varias operaciones relacionadas con la administración de redes.
  - **_NET_BIND_SERVICE_**: Permite hacer _bind_ a un _socket_ de un puerto bien conocido (también conocidos como privilegiados, < 1024).
  - **_SYS_TIME_**: Permite modificar el reloj del sistema.

- Algunas de estas _capabilities_ están habilitadas por defecto en el _container runtime_. Por ejemplo, podemos ver las _capabilities_ habilitadas por defecto en [CRI-O v1.30](https://github.com/cri-o/cri-o/blob/v1.30.0/internal/config/capabilities/capabilities_linux.go#L15-L27) o en [ContainerD v1.7](https://github.com/containerd/containerd/blob/release/1.7/oci/spec.go#L115-L132).
- Las _capabilities_ habilitadas por defecto a través del runtime se asignan a cada contenedor en caso de que el usuario no añada o quite _capabilities_ adicionales.
- Conocer qué _capabilities_ son necesarias para una aplicación require que el desarrollador tenga conocimientos respecto a qué operaciones privilegiadas necesita su aplicación. No hay una herramienta mágica que te diga qué capabilities son necesarias para tu aplicación.

---

## Linux _Capabilities_ - _Thread Capability Sets_
<!-- .slide: style="text-align: left; font-size: 18px;"> -->

Como hemos dicho anteriormente, las _capabilities_ son un atributo de cada hilo, por lo tanto, cada hilo tiene los siguientes _capabilities sets_ que contienen cero o más _capabilities_.

<table align="left">
<tr>
<td><b>Permitted Set</b></td>
<td>
Superset que limita las <em>effective capabilities</em> que un hilo puede asumir. También limita las <em>capabilities</em> que pueden ser añadias al <em>inheritable set</em> por un hilo que tenga la <em>capability</em> <b>SETCAP</b> en su <em>effective set</em>. Si un hilo elimina una <em>capability</em> de su <em>permitted set</em>, no podrá volver adquirir esa <em>capability</em> a no ser que la obtenga mediante una llamada <em>execve</em> a un binario con <em>SETUID</em> o que llame a un binario que tenga esa <em>capability</em> como <em>permitted</em> en su <em>file capability set</em>.
</td>
</tr>
<tr>
<td><b>Inheritable Set</b></td>
<td>
<em>Capabilities</em> que se conservan a través de una llamada <em>execve</em>. Las <em>inherited capabilities</em> permanecen heredables cuando ejecutamos cualquier programa, y son añadidas al <em>permitted set</em> cuando ejecutamos un programa que tiene una de las <em>capabilities</em> presentes en este set también como una <em>inheritable capability</em> a nivel de <em>file capability</em>. Hay que tener en cuenta, que las <em>inheritable capabilities</em> no se preservan generalmente a través de una llamada <em>execve</em> cuando el programa se ejecuta como no-root. Para estos casos de uso, hay que usar <em>ambient capabilities</em>.
</td>
</tr>
<tr>
<td><b>Effective Set</b></td>
<td>
Lista de <em>capabilities</em> utilizada por el Kernel para realizar comprobaciones de permisos para el hilo.
</td>
</tr>
<tr>
<td><b>Bounding Set</b></td>
<td>
Mecanismo utilizado para limitar qué <em>capabilities</em> se pueden ganar durante una llamada <em>execve</em>.
</td>
</tr>
<tr>
<td><b>Ambient Set</b></td>
<td>
<em>Capabilities</em> que son preservadas a través de una llamada <em>execve</em> desde un hilo que no es privilegiado. Para que una <em>capability</em> pueda ser <em>ambient</em>, requiere de estar presente tanto en el <em>permitted</em> como en el <em>inheritable</em> set. Al ejecutar un programa que cambia el UID o GID (gracias al SETUID/SETGID) o al ejecutar un programa que tiene <em>file capabilities</em> el <em>ambient set</em> se limpia automaticamente. Las <em>ambient capabilities</em> se añaden al <em>permitted</em> y al <em>effective</em> set cuando el programa utiliza la llamada <em>execve</em>.
</td>
</tr>
</table>

---

## Linux _Capabilities_ - _File Capability Sets_
<!-- .slide: style="text-align: left; font-size: 18px;"> -->

Además de las _thread capabilities_, también tenemos las _file capabilities_, las cuales son asignadas a un fichero ejecutable el cual obtendrá dichas _capabilities_ en su hilo cuando se ejecute. Estas _capabilities_ se almacenan usando los atributos extendidos de los ficheros. Hay tres _capability sets_ que podemos configurar.

<table align="left">
<tr>
<td><b>Permitted Set</b></td>
<td>
<em>Capabilities</em> permitidas para el hilo, independientemente de las definidias a nivel de <em>inheritable capabilities</em> del hilo.
</td>
</tr>
<tr>
<td><b>Inheritable Set</b></td>
<td>
<em>Capabilities</em> a las cuales se le aplica una regla <em>AND</em> junto con las <em>inheritable capabilities</em> del hilo para determinar cuáles de las <em>inheritable capabilities</em> se habilitan en el <em>permitted set</em> del hilo después del <em>execve</em>.
</td>
</tr>
<tr>
<td><b>Effective Set</b></td>
<td>
Esto no es un <em>set</em> en sí mismo, en su lugar es solo un bit. Si está habilitado, durante una llamada <em>execve</em> todas las <em>capabilities</em> que forman parte del <em>permitted set</em> del hilo se añaden al <em>effective set</em>. En caso contrario, después del <em>execve</em>, ninguna de las <em>capabilities</em> del <em>permitted set</em> serán añadidas al <em>effective set</em>. Cuando habilitamos una <em>file capability</em> en el <em>effective set</em>, esta <em>capability</em> se añadirá de manera automática al <em>permitted set</em> del hilo.
</td>
</tr>
</table>

---

## Demo

### _Capabilities_ &nbsp;en Contenedores

---

# Secure Computing (seccomp)

---

## Secure Computing (seccomp)
<!-- .slide: style="text-align: left; font-size: 18px;"> -->

- Habitualmente los contenedores ejecutan una única aplicación con una lista de tareas bien definidas.

  - Las aplicaciones requieren de acceso a un número limitado de APIs del Kernel del sistema operativo donde se ejecutan. Por ejemplo, un servidor _httpd_ no necesita acceso a la _syscall_ **_mount_**, ¿debería tener acceso a ella?

  - A la hora de limitar los vectores de ataque de un posible proceso comprometido ejecutándose en un contenedor, podemos limitar a qué _syscalls_ tiene acceso el mismo.

- En Kubernetes todo se ejecuta como _unconfined_ (todas las syscalls disponibles) por defecto.

  - En Kubernetes 1.27 hay una funcionalidad en Kubelet que puede utilizarse para aplicar un perfil seccomp por defecto a los workloads que no definen uno específicamente.

- Crear tus propios perfiles _seccomp_ puede ser tedioso y habitualmente requiere de un gran conocimiento de la aplicación. Por ejemplo, el desarrollador tiene que ser consciente de que el framework que utiliza y que le permite escuchar en un puerto específico, por debajo, está lanzando las _syscalls_ _socket_, _bind_ y _listen_.

  - Hay herramientas, como el _oci-seccomp-bpf-hook_, para obtener el listado de _syscalls_ usadas por un proceso.
  
  - Cuando se usan este tipo de herramientas para crear perfiles de uso en contenedores, hay que utilizar el mismo _runtime_ para generarlas que para ejecutarlas. Por ejemplo: _crun_ vs _runc_.

---

## Secure Computing (seccomp)
<!-- .slide: style="text-align: left; font-size: 18px;"> -->

- Ejemplo de perfil seccomp:

<pre><code data-line-numbers="2|3-7|8|10-15|16">{
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "names": [
                "accept4",
                "epoll_wait",
                "pselect6",
                "futex"
            ],
            "action": "SCMP_ACT_ALLOW"
        }
    ]
}
</pre></code>

- Hay 3 posibles acciones que podemos utilizar:

  - **_SCMP_ACT_ALLOW_**: Permite el uso de las _syscalls_ listadas.
  - **_SCMP_ACT_ERRNO_**: Bloquea el uso de las _syscalls_ listadas.
  - **_SCMP_ACT_LOG_**: Permite el uso de cualquier _syscall_, pero notifica aquellas que se han utilizados in estar explícitamente permitidas.

---

## Demo

### _Seccomp_ &nbsp;en Contenedores

---

# Linux _Capabilities_ &nbsp;en Kubernetes

---

## Linux _Capabilities_ &nbsp;en Kubernetes
<!-- .slide: style="text-align: left; font-size: 18px;"> -->

- Actualmente hay algunas limitaciones en Kubernetes cuando nuestras cargas de trabajo no se ejecutan con UID 0:

  - Las _ambient capabilities_ no están soportadas en Kubernetes. Está previsto que se solucione en el futuro: [KEP](https://github.com/kubernetes/enhancements/issues/2763).

- Podemos utilizar _User Namespaces_:

  - Kubernetes v1.30 soporta en modo beta los pods en modo _user namespaces_. [Anuncio de la funcionalidad](https://kubernetes.io/blog/2024/04/22/userns-beta/).
  
  - CRI-O tiene soporte para dicho modo (sólo mediante crun) y ContainerD planea añadirlo de manera estable en la v2.0, por ahora está en fase experimental en la v1.7.

- Ejecutar nuestros contenedores con un UID fijo (como 0) tiene implicaciones de seguridad:

  - Todos los procesos que compartan dicho UID y almacenamiento podrán leer y escribir ficheros en dicho almacenamiento.

  - Si puedes escapar del confinamiento del contenedor a través de una vulnerabilidad (potencialmente en el _runtime_), podrás ver todos los procesos y ficheros en el host cuyo propietario es el UID con el que te ejecutas. Sistemas como SELinux ayudan a reducir la superficie de ataque en el nodo.

- Mientras se añade soporte para las _ambient capabilities_ en Kubernetes, podemos hacer uso de las _file capabilities_ o de aplicaciones _capability aware_.

---

## Linux _Capabilities_ &nbsp;en Kubernetes - Escalación de privilegios
<!-- .slide: style="text-align: left; font-size: 18px;"> -->

- Cuando se usan _file capabilities_ o aplicaciones _capability aware_ en contenedores ejecutándose con un UID no-root, dichos contenedores necesitan poder realizar escaladas de privilegios para obtener dichas _capabilities_.

- Para controlar si un contenedor puede o no realizar escaladas de privilegios existe el parámetro _pod.spec.containers.securityContext.allowPrivilegeEscalation_:

  - Este parámetro controla si un proceso puede obtener más privilegios que su proceso padre. Este parámetro se traduce en la configuración del boleano _no_new_privs_ para el proceso.

  - _AllowPrivilegeEscalation_ se habilitará automáticamente en caso de que el contenedor se ejecute en modo privilegiado o en caso de que tenga acceso a la _capability_ _CAP_SYS_ADMIN_.

- La propuesta de diseño para añadir _no_new_privs_ a Kubernetes puede verse [aquí](https://github.com/kubernetes/design-proposals-archive/blob/main/auth/no-new-privs.md)

- Los procesos con el boleano _no_new_privs_ activado:

  - El proceso o sus hijos no pueden obtener privilegios adicionales.

  - No pueden deshabilitar el bit _no_new_privs_ una vez activado.

  - No pueden cambiar su UID/GID u obtener nuevas _capabilities_, incluso cuando se usan binarios con _setuid_ o _file capabilities_.

  - Módulos de seguridad, tales como SELinux, solo podrán transicionar a tipos de procesos con menos privilegios. Más información [aquí](https://danwalsh.livejournal.com/78312.html).

---

## Demo

### _Capabilities_ &nbsp;en Kubernetes

---

# _Seccomp_ &nbsp;en Kubernetes

---

## _Seccomp_ &nbsp;en Kubernetes
<!-- .slide: style="text-align: left; font-size: 18px;"> -->

- Por defecto, Kubelet buscará perfiles _seccomp_ en la ruta _/var/lib/kubelet/seccomp_. Esta ruta puede ser configurada en el fichero de configuración de Kubelet.

- Múltiples perfiles _seccomp_ pueden coexistir en el mismo directorio.

- Recordad que Kubernetes ejecuta todo como _unconfined_ por defecto.

  - Si utilizamos la funcionalidad de perfil _seccomp_ por defecto, esta funcionalidad trae un perfil _seccomp_ que se utilizará por defecto en caso que el usuario no especifique uno.

---

## Demo

### _Seccomp_ &nbsp;en Kubernetes

---

# Gestionando _Capabilities_ &nbsp;y _Seccomp_ &nbsp;en _Kubernetes_

---

## Gestionando _Capabilities_ &nbsp;y _Seccomp_ &nbsp;en _Kubernetes_
<!-- .slide: style="text-align: left; font-size: 18px;"> -->

- Durante esta presentación hemos utilizado _capabilities_ y _seccomp_ sin tener en cuenta ningún tipo de permiso para utilizar dichas funcionalidades. En el mundo real, queremos poder restringir qué usuarios tienen acceso a qué _capabilities_ o a qué perfiles _seccomp_.

- Los _PodSecurityAdmission_ de Kubernetes pueden utilizarse para controlar esto:

  - Están habilitados por defecto y son el reemplazo de los antiguos y ya obsoletos _Pod Security Policies_.
  
  - Define una serie de [_Pod Security Standards_](https://kubernetes.io/docs/concepts/security/pod-security-standards) que definen qué configuraciones pueden realizarse a nivel de _SecurityContext_ en nuestros contenedores.

    - Una limitación es que estos perfiles son predefinidos y no pueden modificarse. Si se necesita de controles más especificos hay que recurrir a herramientas de terceros como OPA Gatekeeper o Kyverno (entre otras).

---

## Demo

### Gestionando _Capabilities_ &nbsp;y _Seccomp_ &nbsp;en _Kubernetes_

---

# Recursos
<!-- .slide: style="text-align: left; font-size: 18px;"> -->

- [https://linuxera.org/container-security-capabilities-seccomp/](https://linuxera.org/container-security-capabilities-seccomp/)
- [https://linuxera.org/capabilities-seccomp-kubernetes/](https://linuxera.org/capabilities-seccomp-kubernetes/)
- [https://linuxera.org/working-with-pod-security-standards/](https://linuxera.org/working-with-pod-security-standards/)
- [https://man7.org/linux/man-pages/man7/capabilities.7.html](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [Scripts demos](https://tbd.tbd)

---

# Preguntas frecuentes
<!-- .slide: style="text-align: left; font-size: 18px;"> -->

- ¿Por qué las _capabilities_ que obtiene un contenedor en Kubernetes son diferentes de las que obtiene en Podman?
- &shy;<!-- .element: class="fragment" data-fragment-index="1" --> ¿Qué es un programa _capability aware_?
- &shy;<!-- .element: class="fragment" data-fragment-index="2" --> ¿Cual es la formula o algoritmo utilizado para calcular las capabilities durante una llamada _execve_?
<ul style="list-style-type: none;">&shy;<!-- .element: class="fragment" data-fragment-index="3" --> <img src="images/caps2.png" height="290px" width="360px" style="float: center"/></ul>

---

# ¿Preguntas?

---

# ¡Gracias!

<center>Link a esta presentación</center>
<img src="images/slides-qr.png" style="float: center"/>

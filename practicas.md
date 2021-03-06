### CASOS PRACTICOS

Realizaremos algunos ejemplos para tener soltura con nginx y aprender sus principales funcionalidades

NOTA: DESPUES DE REALIZAR CAMBIOS PARA LAS CONFIGURACIONES DEBEMOS REINICIAR EL SERVICIO.

**1- Versión de Nginx instalado.**

    Entrada:
      nginx -v

    Salida:
      nginx version: nginx/1.14.2
    
**2- Servicio Asociado**

    systemctl status nginx

**3- Ficheros de configuración**

   Aqui encontraremos todos los ficheros de configuracion para nginx ```/etc/nginx```. El fichero mas genérico, o mas globalizado por asi decirlo (que su configuracion va a estar en todas las paginas) es ```nginx.conf```.


**4- Pagina web por defecto**

   - Vamos a cambiar la pagina web por defecto.
    
   ```cp /var/www/html/index.html /var/www/html/index.html.ORIGINAL``` --> Realizamos una copia del index para no perderlo
   
   ```echo "<h2> Bienvenidos a Mi servidor web nginx</h2>" > /var/www/html/index.html``` --> Codigo simple de html 
   
   - Recargamos la pagina web y...
    
   ![img](https://i.imgur.com/QWIKP7X.png)
   
   
**5- Virtual Hosting**
  
  - Tendremos dos sitios virtuales con la misma ip y el mismo puerto, por lo tanto, configuraremos cada web por un dominio (www.web1.org y www.web2.org). Lo primero que tenemos que hacer es crear dos carpetas a la misma altura de `/var/www/html`, por lo tanto haremos lo siguiente:

    ```mkdir /var/www/html/web1 /var/www/html/web2```
 
    - En ellas vamos a introducir un index.html basico:
    
    ```echo "<h2>Esta es mi web1</h2>" > /var/www/web1/index.html```
    
    ```echo "<h2>Esta es mi web2</h2>" > /var/www/web2/index.html```
    
    - Copiamos el fichero `/etc/nginx/sites-available/default` para configurar las nuevas páginas por dominio y creamos un enlace duro en `/etc/nginx/sites-enabled`
    
    ```cp /etc/nginx/sites-available/default /etc/nginx/sites-available/web1```
    ```cp /etc/nginx/sites-available/default /etc/nginx/sites-available/web2```
    
    `ln -s /etc/nginx/sites-available/web1 /etc/nginx/sites-enabled/` --> Creamos el enlace duro de web1
    `ln -s /etc/nginx/sites-available/web2 /etc/nginx/sites-enabled/` --> creamos el enlace duro de web2
    
    - Configuramos los ficheros `/etc/nginx/sites-available/web*` que queden tal que así.
    
    [Fichero web1](https://github.com/sergiolaguens/nginx/blob/main/web1) 
    
    [Fichero web2](https://github.com/sergiolaguens/nginx/blob/main/web2)
    
    - Añadimos los dns en el equipo fisico:
    
    `echo "192.168.2.57	www.web1.org" >> /etc/hosts`
    
    `echo "192.168.2.57	www.web2.org" >> /etc/hosts`
    
    - Ahora solo tenemos que buscar www.web1.org y www.web2.org
    
    
**6- Control de Acceso: A la web1 se podrá acceder externa e internamente, a la web2, solo internamente.**

  - Editamos el fichero `/etc/nginx/sites-available/web2` añadiendo las siguientes lineas
  
                location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                try_files $uri $uri/ =404;
                allow 192.168.3.0/24;
                deny all;
        }

  - COMPROBACIONES:
  
    - [Red Interna](https://i.imgur.com/U3MrxVI.png)
    
    - [Red Externa](https://i.imgur.com/oVCvRgs.png)


**7- Autenticacion de Usuarios: Crearemos un directorio en web1 que se llame privado y que solo puedan acceder los usuario válidos.**

  - Creamos el direcotrio en `/var/www/web1` 
  
        mkdir privado
        echo "<h2>Este es el directorio PRIVADO</h2>" > privado/index.html
        
  - Añadimos las siguientes lineas dentro de `/etc/nginx/sites-available/web1`
        
        location /privado {
                auth_basic               "Acceso restringido";
                auth_basic_user_file     /etc/nginx/.htpasswd;
        }
        
  - Instalamos el paquete apache2-utils
  
        sudo apt install apache2-utils
        
  - Creamos las credenciales en el fichero
  
       `cd /etc/nginx`
       `htpasswd -c -m .htpasswd admin` --> Donde admin es el usuario y .htpasswd es el fichero que hemos creado
       
  - COMPROBACIONES:
  
    - [Acceso a www.web1.org y a www.web1.org/privado](https://i.imgur.com/l8vrKvX.png)
    
**8- Red externa pide autenticacion al directorio privado, red interna no.**

  - Agregamos las siguientes lineas al código anterior:
  
        satisfy any;
        allow 192.168.3.0/24;
        deny all;
        
  - COMPROBACIONES: 
  
    - [Red Externa](https://i.imgur.com/Zuom2KH.png)
    
    - [Red Interna](https://i.imgur.com/Ovm2C8J.png)
    
**9- Configurar www.web1.org como acceso seguro.**

  - Instalamos openssl
  
        sudo apt install openssl
        
  - Generamos clave privada para www.web1.orgssl on;
	ssl_certificate /etc/ssl/web1.crt;
	ssl_certificate_key /etc/ssl/web1.key;

  
       `cd /etc/ssl` --> Venimos aqui porque será donde guardemos las claves. Puede ser cualquier directorio pero lo haré aqui para tenerlo organizado
       `openssl genrsa -out web1.key 2048`
       
  - Generamos el Certificado
  
        openssl req -new -key web1.key -out web1.csr
        
  - Firmamos el certificado
  
        openssl x509 -req -days 365 -in web1.csr -signkey web1.key -out web1.crt
        
        
  - Copiamos web1 para configurarlo como ssl:
  
        cd /etc/nginx/sites-available
        cp web1 web1-ssl
        ln -s /etc/nginx/sites-availabe/web1-ssl /etc/nginx/sites-enabled/
        
  - Añadimos las siguientes lineas a `/etc/nginx/sites-available/web1-ssl`
  
        ssl on;
	    ssl_certificate /etc/ssl/web1.crt;
	    ssl_certificate_key /etc/ssl/web1.key;
     
  - Reiniciamos el servicio
   
        systemctl restart nginx
        
        
  - Por último buscamos en el navegador https://www.web1.org y fin. Tenemos nuestro sitio web seguro. Aunque en el certificado de https nos ponga no seguro es porque lo hemos firmado nostros. Esto lo mas adecuado es hacerlo a través de una empresa certificadora.


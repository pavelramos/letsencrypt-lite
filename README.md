<h2>Let's Encrypt utilizando acme-tiny</h2>

Teniendo en cuenta la gran cantidad de recursos que utiliza Certbot, el cliente oficial de Let's Encrypt, es necesaria una alternativa ligera que permita configurar certificados SSL para las páginas web  en nuestros servidores con pocos recursos. Para esto usaremos un script escrito en tan sólo 200 líneas de código, el sorprendente  [acme-tiny](https://github.com/diafygi/acme-tiny).

Este es el procedimiento en un 128MB OpenVZ que utilizo para alojar una página web de poco tráfico en donde no es posible crear un swap para apoyar a la memoria.

1.  Generamos la clave privada para nuestra cuenta en Let's Encrypt:

		openssl genrsa 4096 > /etc/letsencrypt/cuentas/cuenta.key

2. Generamos la clave privada para el dominio:

        openssl genrsa 4096 > /etc/letsencrypt/certs/dominio.com/dominio.com.key

3. Ahora con esta clave crearemos nuestra solicitud de firma de certificado (CSR)

	Para un solo dominio:

    	openssl req -new -sha256 -key /etc/letsencrypt/certs/dominio.com.key -subj "/CN=dominio.com" > /etc/letsencrypt/csr/dominio.com/dominio.com.csr
    
	Para múltiples dominios (en caso querramos usar dominio.com y www.dominio.com, por ejemplo):

		openssl req -new -sha256 -key /etc/letsencrypt/certs/dominio.com/dominio.com.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:dominio.com,DNS:www.dominio.com")) > /etc/letsencrypt/csr/dominio.com/dominio.com.csr
        
4. Para que el certificado sea firmado por Let's Encrypt, es necesario que se genere un archivo de comprobación de forma temporal y que se pueda acceder a él mediante HTTP agregando ".well-known/acme-challenge/" al enlace del dominio, para ello crearemos una carpeta dentro de www, en donde se crearán esos archivos temporales:

		mkdir -p /var/www/validar/
        
	Hay que configurar nuestro servidor web para que se pueda acceder vía HTTP desde la carpeta que creamos hacia nuestro dominio. En nginx la configuración sería de esta manera:
    
    	location ^~ /.well-known/acme-challenge/ {
       		alias /var/www/validar/;
		}

5. En este paso utilizaremos acme-tiny para firmar el certificado, dependiendo de dónde hayamos guardado el archivo acme_tiny.py, ejecutaremos lo siguiente:

		python acme_tiny.py --account-key /etc/letsencrypt/cuentas/cuenta.key --csr /etc/letsencrypt/csr/dominio.com/dominio.com.csr --acme-dir /var/www/validar/ > /etc/letsencrypt/certs/dominio.com/dominio.com.crt
        
6. Ahora sólo falta descargar el certificado intermediario de Let's Encrypt (CA), esto se realiza con:

		wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > /etc/letsencrypt/certs/dominio.com/intermediario.pem
        
7. Hasta aquí ya se podrá activar el HTTPS en nuestros servidores web, para el caso de nginx, como el mío, hay que hacer unos cuantos pasos más, el primero será de añadir el certificado intermediario al firmado (dominio.com.crt) de esta manaera:

		# Nos dirijimos a la carpeta de nuestro dominio dentro de /certs/
        cd /etc/letsencrypt/certs/dominio.com/
        # Ahora unimos nuestros certificados en un certificado final
        cat dominio.com.crt intermediario.pem > dominio.com.pem
        
8. Todo listo para configurar nginx, configuramos la conexión HTTPS vía el puerto 443 en nuestra configuración, o en mi caso en el archivo /var/www/dominio.com/conf/nginx/ssl.conf
		
        listen 443 ssl http2;
		ssl on;
		ssl_certificate     	/etc/letsencrypt/certs/dominio.com/dominio.com.pem;
		ssl_certificate_key     /etc/letsencrypt/certs/dominio.com/dominio.com.key;
        
9. Por último vamos a forzar la conexión HTTPS para nuestro dominio en /etc/nginx/conf.d/force-ssl-domain.com.conf

		server {
        listen 80;
        server_name www.dominio.com dominio.com;
        return 301 https://dominio.com$request_uri;
        
10. Reiniciamos nginx

		sudo service nginx reload
        
¡Eso es todo!

Es muy importante siempre utilizar un usuario con permisos limitados para editar dentro de la carpeta /var/www/validar/ y gestionar los certificados. No realizar estos pasos desde el usuario root.

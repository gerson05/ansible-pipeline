# ansible-pipeline — Resumen de cambios y uso

Proyecto para desplegar Jenkins + SonarQube + Postgres con Nginx proxy usando Ansible y Docker Compose.

Integrantes:
- Alejandro Muñoz
- Gerson Hurtado

## Cambios aplicados en el repo
- playbook.yml
  - Se creó/actualizó el playbook con dos plays:
    - hosts: nginx
      - Instala y configura Nginx.
      - Copia plantilla `templates/nginx-base.conf.j2` a `/etc/nginx/nginx.conf`.
      - Sincroniza la app (carpeta `../Teclado/`) al host web con `synchronize`.
      - Habilita y arranca el servicio y notifica handler para reiniciar nginx.
    - hosts: jenkins
      - Instala Docker y docker-compose.
      - Copia `Dockerfile.jenkins`, `plugins.txt` y `docker-compose.yml` al host remoto.
      - Ejecuta `docker-compose down`, `docker-compose build jenkins` y `docker-compose up -d`.
      - Instala plugins en Jenkins ejecutando:
        `docker exec -i jenkins jenkins-plugin-cli --plugin-file - < /home/{{ ansible_user }}/plugins.txt`
      - Espera disponibilidad de Jenkins y SonarQube con `wait_for`.
  - `gather_facts: true` habilitado para usar variables como `ansible_lsb.codename`.
  - Handlers añadidos para reiniciar docker/jenkins.

- docker-compose.yml
  - Servicio `jenkins` definido con `build` (Dockerfile.jenkins) y `image: jenkins-custom:latest`.
  - Se mantiene `container_name: jenkins` (compatibilidad con tareas que hacen `docker exec`).
  - Políticas `restart: unless-stopped`, volúmenes definidos y mapeos de puertos:
    - `"80:8080"`, `"8443:8443"`, `"50000:50000"`.
  - Servicios `sonarqube` y `db` (Postgres) añadidos con volúmenes y mapeos (`9000:9000`).

- plugins.txt
  - Lista de plugins base recomendados para Jenkins (archivo copiado al host remoto y usado para la instalación).
  - Recomendación: fijar versiones en lugar de `:latest` para entornos reproducibles.

- inventory.ini
  - Archivo con grupos `jenkins` y `nginx`. Contiene placeholders `JENKINS_IP`, `NGINX_IP` y credenciales (reemplazar por valores reales).

## Archivos importantes
- `playbook.yml` — flujo de despliegue completo.
- `docker-compose.yml` — stack Docker.
- `plugins.txt` — plugins a instalar.
- `Dockerfile.jenkins` — Dockerfile personalizado (debe existir en repo).
- `inventory.ini` — inventario Ansible (reemplazar IPs/credenciales).
- `vars/main.yml` — archivo de variables requerido (ver ejemplo abajo).
- `templates/nginx-base.conf.j2` — plantilla Nginx (asegurar que exista).

## Variables mínimas necesarias (vars/main.yml) — ejemplo
```yaml
# filepath: c:\Users\alejo\Documents\SEMESTRE VIII\ingesoft 5\ansible-pipeline\ansible-pipeline\vars\main.yml
ansible_user: adminuser
jenkins_port: 80
sonar_port: 9000
nginx_host: 20.168.193.87
jenkins_host: "{{ ansible_host }}"
sonar_host: "{{ ansible_host }}"
```

## Pasos para ejecutar (desde la máquina de control)
1. Rellenar `inventory.ini` con las IP reales y credenciales.
2. Crear/editar `vars/main.yml` con las variables necesarias.
3. Ejecutar el playbook:
   ```bash
   ansible-playbook -i inventory.ini playbook.yml -vv
   ```
4. Verificar contenedores en la VM Jenkins:
   ```bash
   ssh adminuser@JENKINS_IP
   sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
   sudo docker logs -f jenkins
   ```
5. Probar endpoints:
   - Jenkins (por proxy Nginx): http://NGINX_IP/jenkins/
   - SonarQube (por proxy): http://NGINX_IP/sonar/
   - Directo Jenkins: http://JENKINS_IP:{{ jenkins_port }}
   - Directo SonarQube: http://JENKINS_IP:{{ sonar_port }}/sonar/

## Verificaciones útiles (comandos)
- Estado Nginx en host nginx:
  ansible nginx -i inventory.ini -m shell -a "systemctl status nginx --no-pager" -u adminuser --become
- Docker PS en host jenkins:
  ansible jenkins -i inventory.ini -m shell -a "docker ps --format 'table {{.Names}}\t{{.Ports}}'" -u adminuser --become
- Obtener contraseña inicial de Jenkins:
  sudo docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

## Notas y recomendaciones
- Recomendado fijar versiones de plugins en `plugins.txt` en lugar de `:latest`.
- Si el puerto 80 del host está ocupado, cambiar mapeo en `docker-compose.yml` (ej. `"8080:8080"`) y actualizar `vars/main.yml` y `playbook.yml` si es necesario.
- Considerar eliminar `container_name` en `docker-compose.yml` para evitar conflictos de nombres (si se elimina, actualizar las tareas que usan `docker exec jenkins`).
- La instalación de docker-compose se hace con una release fija (1.29.2) — revisar versión si se prefiere una más reciente.
- Comprueba que `templates/nginx-base.conf.j2` redirija correctamente `/jenkins` y `/sonar` al host/puerto adecuados.

## Problemas comunes
- Conflicto de nombres de contenedor: ejecutar `docker rm -f jenkins` en la VM si hay un contenedor previo.
- Puerto en uso (ej. 50000): cambiar mapeo en compose o liberar puerto.
- Warning `version: "3"` en docker-compose: la CLI moderna ignora `version` — puedes eliminarla si lo deseas.

Si quieres, aplico automáticamente este README.md en el workspace. ¿Lo guardo ahora?# filepath: c:\Users\alejo\Documents\SEMESTRE VIII\ingesoft 5\ansible-pipeline\ansible-pipeline\README.md

# ansible-pipeline — Resumen de cambios y uso

Proyecto para desplegar Jenkins + SonarQube + Postgres con Nginx proxy usando Ansible y Docker Compose.

Integrantes:
- Alejandro Muñoz
- Gerson Hurtado

## Cambios aplicados en el repo
- playbook.yml
  - Se creó/actualizó el playbook con dos plays:
    - hosts: nginx
      - Instala y configura Nginx.
      - Copia plantilla `templates/nginx-base.conf.j2` a `/etc/nginx/nginx.conf`.
      - Sincroniza la app (carpeta `../Teclado/`) al host web con `synchronize`.
      - Habilita y arranca el servicio y notifica handler para reiniciar nginx.
    - hosts: jenkins
      - Instala Docker y docker-compose.
      - Copia `Dockerfile.jenkins`, `plugins.txt` y `docker-compose.yml` al host remoto.
      - Ejecuta `docker-compose down`, `docker-compose build jenkins` y `docker-compose up -d`.
      - Instala plugins en Jenkins ejecutando:
        `docker exec -i jenkins jenkins-plugin-cli --plugin-file - < /home/{{ ansible_user }}/plugins.txt`
      - Espera disponibilidad de Jenkins y SonarQube con `wait_for`.
  - `gather_facts: true` habilitado para usar variables como `ansible_lsb.codename`.
  - Handlers añadidos para reiniciar docker/jenkins.

- docker-compose.yml
  - Servicio `jenkins` definido con `build` (Dockerfile.jenkins) y `image: jenkins-custom:latest`.
  - Se mantiene `container_name: jenkins` (compatibilidad con tareas que hacen `docker exec`).
  - Políticas `restart: unless-stopped`, volúmenes definidos y mapeos de puertos:
    - `"80:8080"`, `"8443:8443"`, `"50000:50000"`.
  - Servicios `sonarqube` y `db` (Postgres) añadidos con volúmenes y mapeos (`9000:9000`).

- plugins.txt
  - Lista de plugins base recomendados para Jenkins (archivo copiado al host remoto y usado para la instalación).
  - Recomendación: fijar versiones en lugar de `:latest` para entornos reproducibles.

- inventory.ini
  - Archivo con grupos `jenkins` y `nginx`. Contiene placeholders `JENKINS_IP`, `NGINX_IP` y credenciales (reemplazar por valores reales).

## Archivos importantes
- `playbook.yml` — flujo de despliegue completo.
- `docker-compose.yml` — stack Docker.
- `plugins.txt` — plugins a instalar.
- `Dockerfile.jenkins` — Dockerfile personalizado (debe existir en repo).
- `inventory.ini` — inventario Ansible (reemplazar IPs/credenciales).
- `vars/main.yml` — archivo de variables requerido (ver ejemplo abajo).
- `templates/nginx-base.conf.j2` — plantilla Nginx (asegurar que exista).

## Variables mínimas necesarias (vars/main.yml) — ejemplo
```yaml
ansible_user: adminuser
jenkins_port: 80
sonar_port: 9000
nginx_host: 20.168.193.87
jenkins_host: "{{ ansible_host }}"
sonar_host: "{{ ansible_host }}"
```

## Verificaciones útiles (comandos)
- Estado Nginx en host nginx:
  ansible nginx -i inventory.ini -m shell -a "systemctl status nginx --no-pager" -u adminuser --become
- Docker PS en host jenkins:
  ansible jenkins -i inventory.ini -m shell -a "docker ps --format 'table {{.Names}}\t{{.Ports}}'" -u adminuser --become
- Obtener contraseña inicial de Jenkins:
  sudo docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword


---
title: <span style="color:#c0c0c0">GitHub</span> <span style="color:white">Pages</span>
layout: single
excerpt: ""
header:
show_date: true
classes: wide
header:
  teaser: "/assets/images/githubpages/GPBanner.png"
  teaser_home_page: true
  icon: ""
categories:
  - Tutoriales
tags:
  - GitHub Pages
  - Jekyll
  - Ruby
  - Bundler
  - Desarrollo web
  - Markdown


date: 2024-08-28 
---

# <span style="color:#2D9CDB">CREAR PÁGINA CON GITHUB PAGES</span>

![](/assets/images/githubpages/GPBanner.png)

GitHub Pages es un servicio ofrecido por GitHub que permite a los usuarios alojar sitios web estáticos directamente desde un repositorio de GitHub.

Puedes crear un repositorio nuevo en GitHub para tu sitio web. El <u>nombre del repositorio</u> debe seguir ciertas convenciones si quieres usar GitHub Pages para tu usuario o tu organización (<span style="color:yellow">`username.github.io`</span> u <span style="color:yellow">`organization.github.io`</span>). 


Para usar mi repositorio puedes hacer un fork del mismo para crear una copia en tu propia cuenta de GitHub.

- Ve a: *[https://github.com/0so/0so.github.io](https://github.com/0so/0so.github.io)*
- Haz clic en el botón "Fork" en la esquina superior derecha de la página del repositorio.
![](/assets/images/githubpages/1.png)
- Selecciona tu cuenta de GitHub donde deseas crear el fork.
- Espera a que se complete el fork y configura el nombre del repositorio.



## <span style="color:#27AE60">Desplegar entorno</span>

### <span style="color:#F39C12">Tecnologías utilizadas</span>

- **Jekyll**: Generador de sitios web estáticos que convierte Markdown y HTML en sitios web.
- **Ruby**: Lenguaje de programación utilizado por Jekyll.
- **Bundler**: Herramienta de gestión de dependencias para Ruby.  


### <span style="color:#E74C3C">1. Instalar Bundler</span>



Bundler es una herramienta de gestión de dependencias para Ruby. En este caso, se utiliza para instalar las dependencias necesarias para ejecutar el sitio web.

```bash
sudo gem install bundler -v 2.5.6
```

### <span style="color:#E74C3C">2. Instalar dependencias:



Una vez instalado Bundler, ejecute el siguiente comando:

```bash
sudo bundle install 
``` 

Esto lee el archivo Gemfile en el proyecto, que lista todas las gemas que el proyecto necesita, y las instala.

### <span style="color:#E74C3C">3. Ejecutar Jekyll para iniciar el servidor de desarrollo.



```bash
bundle exec jekyll serve --livereload
```
Inicia un servidor local que sirve el sitio web generado por Jekyll. La opción `--livereload` permite que el servidor actualice automáticamente la vista previa del sitio web cuando realizas cambios en los archivos del proyecto, lo que te permite ver cómo queda tu sitio web en tiempo real mientras trabajas en él. Es muy útil para desarrollo y pruebas antes de desplegarlo en un servidor en producción.
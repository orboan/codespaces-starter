ARG ROOT_CONTAINER=ubuntu:22.04

FROM $ROOT_CONTAINER 

ARG NB_USER="entorn"
ARG NB_UID="1000"
ARG NB_GID="100"

ARG TURBOVNC_VERSION=3.0

ARG ijava_version=1.3.0
ARG java_version=17.0.6
ARG java_provider=amzn
ARG python_major_version=3.10
ARG language=ca_ES

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ENV \
    CONDA_BASE_PATH=/opt/conda \
    CONDA_DIR=/opt/conda \
    IJAVA_VERSION=$ijava_version \
    DOMAIN=pluralcamp.com \
    PYTHON_VERSION_MAJOR=$python_major_version \
    PYTHON_VERSION=$python_major_version \
    LANG="${language}.UTF-8" \
    LC_CTYPE="${language}.UTF-8" \
    LC_ALL="${language}.UTF-8" \
    LANGUAGE="${language}:ca" \
    REMOVE_DASH_LINECOMMENT=true \
    SHELL=/bin/bash \
    USER="${NB_USER}" \
    USER_GID="${NB_GID}" \
    NB_USER="${NB_USER}" \
    NB_UID="${NB_UID}" \
    NB_GID="${NB_GID}" \
    XDG_CACHE_HOME="${HOME}/.cache/" \
    XDG_RUNTIME_DIR="/tmp" \
    DISPLAY=":1" \
    TERM="xterm" \
    DEBIAN_FRONTEND="noninteractive" \
    RESOURCES_PATH="/resources" \
    SSL_RESOURCES_PATH="/resources/ssl" \
    JUPYTER_ENABLE_LAB=yes \
    SDKMAN_DIR="/opt/sdkman" \
    NOTEBOOK_ARGS="--notebook-dir=/home/$NB_USER --no-browser --NotebookApp.password='sha1:c4f869f064af:98fd6cb811e7e05beb33167b272554336c712f14'"

ENV PATH="${CONDA_DIR}/bin:${PATH}" \
    HOME="/home/${NB_USER}"
ENV WORKSPACE_HOME="${HOME}" 

USER root

COPY resources/scripts/clean-layer.sh  /usr/bin/clean-layer.sh
RUN \
    chmod a+rwx /usr/bin/clean-layer.sh && \
    mkdir -p $RESOURCES_PATH && chmod a+rwx $RESOURCES_PATH

# Copy a script that we will use to correct permissions after running certain commands
COPY fix-permissions /usr/local/bin/fix-permissions
RUN chmod a+rx /usr/local/bin/fix-permissions

ENV DEBIAN_FRONTEND noninteractive

RUN \
        apt-get -y update; \
        DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends locales; \
        echo "${language}.UTF-8 UTF-8" > /etc/locale.gen; \
        locale-gen; \
        dpkg-reconfigure --frontend=noninteractive locales; \
        update-locale LANG="${language}.UTF-8"; \
        clean-layer.sh

COPY resources/scripts/install-basics.sh  /usr/bin/install-basics.sh
RUN \
    chmod a+rwx /usr/bin/install-basics.sh && \
    install-basics.sh && \
    clean-layer.sh

# Create NB_USER with name jovyan user with UID=1000 and in the 'users' group
# and make sure these dirs are writable by the `users` group.
RUN echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers && \
    sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers && \
    useradd -l -m -s /bin/bash -N -u "${NB_UID}" "${NB_USER}" && \
    mkdir -p "${CONDA_DIR}" && \
    chown "${NB_USER}:${NB_GID}" "${CONDA_DIR}" && \
    chmod g+w /etc/passwd && \
    fix-permissions "${HOME}" && \
    fix-permissions "${CONDA_DIR}"
RUN echo "$NB_USER  ALL=(ALL)  NOPASSWD: ALL" | sudo tee --append /etc/sudoers && \
mkdir -p /workspaces && chown -R $NB_USER:$NB_GID /workspaces

USER ${NB_UID}

# Setup workspaces directory
RUN \
    ln -s /workspaces /home/$NB_USER/workspaces && \
    fix-permissions "/home/${NB_USER}"

# Download and install Micromamba, and initialize Conda prefix.
#   <https://github.com/mamba-org/mamba#micromamba>
#   Similar projects using Micromamba:
#     - Micromamba-Docker: <https://github.com/mamba-org/micromamba-docker>
#     - repo2docker: <https://github.com/jupyterhub/repo2docker>
# Install Python, Mamba and jupyter_core
# Cleanup temporary files and remove Micromamba
# Correct permissions
# Do all this in a single RUN command to avoid duplicating all of the
# files across image layers when the permissions change
COPY --chown="${NB_UID}:${NB_GID}" initial-condarc "${CONDA_DIR}/.condarc"
WORKDIR /tmp
RUN set -x && \
    arch=$(uname -m) && \
    if [ "${arch}" = "x86_64" ]; then \
        # Should be simpler, see <https://github.com/mamba-org/mamba/issues/1437>
        arch="64"; \
    fi && \
    wget -qO /tmp/micromamba.tar.bz2 \
        "https://micromamba.snakepit.net/api/micromamba/linux-${arch}/latest" && \
    tar -xvjf /tmp/micromamba.tar.bz2 --strip-components=1 bin/micromamba && \
    rm /tmp/micromamba.tar.bz2 && \
    PYTHON_SPECIFIER="python=${PYTHON_VERSION}" && \
    if [[ "${PYTHON_VERSION}" == "default" ]]; then PYTHON_SPECIFIER="python"; fi && \
    # Install the packages
    ./micromamba install \
        --root-prefix="${CONDA_DIR}" \
        --prefix="${CONDA_DIR}" \
        --yes \
        "${PYTHON_SPECIFIER}" \
        'mamba' \
        'jupyter_core' && \
    rm micromamba && \
    # Pin major.minor version of python
    mamba list python | grep '^python ' | tr -s ' ' | cut -d ' ' -f 1,2 >> "${CONDA_DIR}/conda-meta/pinned" && \
    mamba clean --all -f -y && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

RUN mamba install --quiet --yes \
    'nodejs' \
    'notebook' \
    'jupyterlab' && \
    jupyter notebook --generate-config && \
    mamba clean --all -f -y && \
    npm cache clean --force && \
    jupyter lab clean && \
    rm -rf "/home/${NB_USER}/.cache/yarn" && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

ENV JUPYTER_PORT=8888
EXPOSE $JUPYTER_PORT

# Configure container startup
CMD ["start-notebook.sh"]

# Copy local files as late as possible to avoid cache busting
COPY start-notebook.sh start-singleuser.sh /usr/local/bin/
# Currently need to have both jupyter_notebook_config and jupyter_server_config to support classic and lab
COPY jupyter_server_config.py docker_healthcheck.py /etc/jupyter/

# Configure container startup
ENTRYPOINT ["tini", "-g", "--"]
#CMD ["start.sh"]

# Copy local files as late as possible to avoid cache busting
COPY start.sh /usr/local/bin/

WORKDIR "${HOME}"

USER root

# Legacy for Jupyter Notebook Server, see: [#1205](https://github.com/jupyter/docker-stacks/issues/1205)
RUN sed -re "s/c.ServerApp/c.NotebookApp/g" \
    /etc/jupyter/jupyter_server_config.py > /etc/jupyter/jupyter_notebook_config.py && \
    fix-permissions /etc/jupyter/

# HEALTHCHECK documentation: https://docs.docker.com/engine/reference/builder/#healthcheck
# This healtcheck works well for `lab`, `notebook`, `nbclassic`, `server` and `retro` jupyter commands
# https://github.com/jupyter/docker-stacks/issues/915#issuecomment-1068528799
HEALTHCHECK --interval=5s --timeout=3s --start-period=5s --retries=3 \
    CMD /etc/jupyter/docker_healthcheck.py || exit 1


### BASICS START ###
# prepare ssh for inter-container communication for remote python kernel
COPY resources/scripts/install-ssh.sh  /usr/bin/install-ssh.sh
RUN \
    chmod a+rwx /usr/bin/install-ssh.sh && \
    install-ssh.sh && \
    # Cleanup
    clean-layer.sh
### BASICS END ###


### START RUNTIMES ###

## Java
ENV \
    JAVA_VERSION=${java_version} \
    JAVA_PROVIDER=${java_provider}
COPY resources/scripts/install-java.sh  /usr/bin/install-java.sh
RUN \
    chmod a+rwx /usr/bin/install-java.sh && \
    install-java.sh && \
    # Cleanup
    clean-layer.sh
    
### END RUNTIMES ###

#### KERNELS START ####

COPY resources/ikernels $RESOURCES_PATH/ikernels/

## java kernel start ##

COPY resources/config/ijava $RESOURCES_PATH/config/ijava/

RUN \
    chmod +x $RESOURCES_PATH/ikernels/java-kernel.sh ;\
    /bin/bash $RESOURCES_PATH/ikernels/java-kernel.sh ;\
    # Cleanup
    clean-layer.sh

## java kernel end ##

## javascript kernel start ##
RUN \
    chmod +x $RESOURCES_PATH/ikernels/tslabs-kernel.sh ;\
    /bin/bash $RESOURCES_PATH/ikernels/tslabs-kernel.sh ;\
    # Cleanup
    clean-layer.sh

## javascript kernel end ##

#### KERNELS END ####

ARG password=pluralcamp

### Setting passwd for NB_USER and root
RUN \
    echo "${NB_USER}:$password" | chpasswd && \
    echo "root:$password" | chpasswd

## Jupyter Extensions
COPY resources/jupyter-extensions/install-basic-jupyter-extensions.sh $RESOURCES_PATH/jupyter-extensions/install-basic-jupyter-extensions.sh

RUN \
    chmod +x $RESOURCES_PATH/jupyter-extensions/install-basic-jupyter-extensions.sh && \
    /bin/bash $RESOURCES_PATH/jupyter-extensions/install-basic-jupyter-extensions.sh && \
    # Cleanup
    clean-layer.sh

USER $NB_USER
## VS Code
## VS Code Server: https://github.com/codercom/code-server -- latest
ENV CODESERVER_VER=4.10.0
COPY resources/ides/vs-code-server.sh $RESOURCES_PATH/ides/vs-code-server.sh
RUN \
    sudo chmod +x $RESOURCES_PATH/ides/vs-code-server.sh && \
    /bin/bash $RESOURCES_PATH/ides/vs-code-server.sh && \
    # Cleanup
    clean-layer.sh

USER root
## DBs

COPY resources/docker $RESOURCES_PATH/docker/
COPY resources/dbs $RESOURCES_PATH/dbs/

########## postgres start
ARG PG=true
#COPY resources/dbs/postgres $RESOURCES_PATH/dbs/postgres/
RUN \
    mkdir -p /home/$NB_USER/.local/bin && chown -R $NB_USER:$NB_GID /home/$NB_USER/.local/bin && \
    if [ "$PG" == "true" ]; then \
        chmod +x $RESOURCES_PATH/dbs/postgres/pg-install.sh ;\
        /bin/bash $RESOURCES_PATH/dbs/postgres/pg-install.sh ;\
    fi \
    # Cleanup
    && clean-layer.sh

######### mysql start
ARG MYSQL=true
#COPY resources/dbs/mysql/ $RESOURCES_PATH/dbs/mysql/
RUN \
    mkdir -p /home/$NB_USER/.local/bin && chown -R $NB_USER:$NB_GID /home/$NB_USER/.local/bin && \
    if [ "$MYSQL" == "true" ]; then \
        chmod +x $RESOURCES_PATH/dbs/mysql/mysql-install.sh ;\
        /bin/bash $RESOURCES_PATH/dbs/mysql/mysql-install.sh ;\
    fi \
    # Cleanup
    && clean-layer.sh

######### mysql end


### GUI TOOLS ###

# Add the defaults from /lib/x86_64-linux-gnu, otherwise lots of no version errors
# cannot be added above otherwise there are errors in the installation of the gui tools
# Call order: https://unix.stackexchange.com/questions/367600/what-is-the-order-that-linuxs-dynamic-linker-searches-paths-in

ENV LD_LIBRARY_PATH=/lib/x86_64-linux-gnu:/usr/lib/x86_64-linux-gnu:$CONDA_ROOT/lib

# Install xfce4 & gui tools (firefox and chrome included)
COPY resources/scripts/install-gui.sh /usr/bin/
RUN \
    chmod a+rwx /usr/bin/install-gui.sh && \
    install-gui.sh \
    # Cleanup
    && clean-layer.sh


## VNC
# Remove light-locker to prevent screen lock
#ARG TURBOVNC=true
#ARG TURBOVNC_VERSION=3.0
COPY resources/tools/turbovnc.sh $RESOURCES_PATH/tools/turbovnc.sh
ENV TURBOVNC_VERSION=${TURBOVNC_VERSION}
ARG TURBOVNC=true
RUN \
    if [ "$TURBOVNC" == "true" ]; then \
        chmod +x $RESOURCES_PATH/tools/turbovnc.sh ;\
        /bin/bash $RESOURCES_PATH/tools/turbovnc.sh ;\
    fi && \
    # Cleanup
    clean-layer.sh

### GUI Tools END ####

## Configurations

COPY resources/scripts/set-jupyter-config.sh resources/scripts/jpy-customize.sh resources/scripts/set-lab-themes.sh /usr/bin/
COPY resources/config $RESOURCES_PATH/config
COPY resources/logos $RESOURCES_PATH/logos

## jupyter customizations
RUN \
    chmod +x \
    /usr/bin/set-jupyter-config.sh \
    /usr/bin/jpy-customize.sh \
    /usr/bin/set-lab-themes.sh \
    && \
    /usr/bin/set-jupyter-config.sh && \
    /usr/bin/jpy-customize.sh && \
    /usr/bin/set-lab-themes.sh && \
# Directory for apps installed by user && \
    mkdir -p /opt/apps && \
    chown $NB_USER:$NB_GID /opt/apps && \
# icons for jupyter-server-proxy && \
    mkdir -p /etc/jupyter/icons

COPY resources/icons /etc/jupyter/icons

COPY resources/config/.bashrc resources/config/.profile /home/$NB_USER/
COPY resources/bin-hooks/start-custom.sh /usr/local/bin/start-notebook.d/

RUN \
        chown $NB_USER:$NB_GID -R /opt/sdkman && \
        find /home/$NB_USER -maxdepth 3 -type d -exec chown $NB_USER:$NB_GID {} \; && \
        find /home/$NB_USER -maxdepth 3 -type f -exec chown $NB_USER:$NB_GID {} \; && \
        mkdir -p /home/$NB_USER/.local/share/jupyter/ && \
        chown entorn:users -R /home/$NB_USER/.local && \
        chmod +x /usr/local/bin/start-notebook.d/*

## Jupyter Server Proxy config
ARG CONFIG_NB_FILE=/etc/jupyter/jupyter_server_config.py
COPY ./proxy-server.entorn-io.py /tmp/
RUN \
        cat /tmp/proxy-server.entorn-io.py >> $CONFIG_NB_FILE && \
        rm /etc/jupyter/jupyter_notebook_config.py && \
        clean-layer.sh

COPY webserver.py /usr/bin/webserver.py

COPY resources/scripts/startup.sh /home/$NB_USER/.local/bin/startup.sh
RUN chmod +x /home/$NB_USER/.local/bin/startup.sh

USER $NB_USER

# Add R mimetype option to specify how the plot returns from R to the browser
COPY --chown=${NB_UID}:${NB_GID} Rprofile.site /opt/conda/lib/R/etc/

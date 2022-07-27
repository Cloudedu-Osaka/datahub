FROM buildpack-deps:focal-scm

ENV CONDA_DIR /opt/conda

# Set up common env variables
ENV TZ=Asia/Tokyo
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

ENV LC_ALL en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US.UTF-8
ENV DEBIAN_FRONTEND=noninteractive
ENV NB_USER jovyan
ENV NB_UID 1000

RUN adduser --disabled-password --gecos "Default Jupyter user" ${NB_USER}

# Create user owned R libs dir
# This lets users temporarily install packages
ENV R_LIBS_USER /opt/r
RUN install -d -o ${NB_USER} -g ${NB_USER} ${R_LIBS_USER}

RUN apt-get -qq update --yes && \
    apt-get -qq install --yes \
            tar \
            vim \
            micro \
            mc \
            tini \
            build-essential \
            locales > /dev/null

RUN echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && \
    locale-gen

# for nbconvert
# FIXME: Understand what exactly we want
# texlive-plain-generic is new name of texlive-generic-recommended
RUN apt-get update > /dev/null && \
    apt-get -qq install --yes \
            pandoc \
            texlive-xetex \
            texlive-fonts-recommended \
            # provides FandolSong-Regular.otf for issue #2714
            texlive-lang-chinese \
            texlive-plain-generic > /dev/null

# Install packages needed by notebook-as-pdf
# Default fonts seem ok, we just install an emoji font
RUN apt-get update && \
    apt-get install --yes \
            libx11-xcb1 \
            libxtst6 \
            libxrandr2 \
            libasound2 \
            libpangocairo-1.0-0 \
            libatk1.0-0 \
            libatk-bridge2.0-0 \
            libgtk-3-0 \
            libnss3 \
            libxss1 \
            fonts-noto-color-emoji > /dev/null && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

USER ${NB_USER}
WORKDIR /home/${NB_USER}

ENV PATH ${CONDA_DIR}/bin:$PATH

USER root

# Install mambaforgeas root - the script chowns to $NB_USER by the end
COPY install-mambaforge.bash /tmp/install-mambaforge.bash
RUN /tmp/install-mambaforge.bash

USER ${NB_USER}

COPY environment.yml /tmp/environment.yml

RUN mamba env update -p ${CONDA_DIR} -f /tmp/environment.yml && mamba clean -afy

COPY infra-requirements.txt /tmp/infra-requirements.txt
RUN pip install --no-cache -r /tmp/infra-requirements.txt

# Set up notebook-as-pdf dependencies
ENV PYPPETEER_HOME ${CONDA_DIR}
RUN pyppeteer-install

EXPOSE 8888

# Temporarily install newer version of jupyterlab-link-share
# Move this back to just installing off infra-requirements once we are a bit stable
RUN pip install -U jupyterlab-link-share==0.2.4
RUN pip install traitlets==5.1.0
RUN pip install nose
RUN pip install solarsystem

RUN jupyter contrib nbextension install --sys-prefix
RUN jupyter nbextension enable hide_input/main
RUN jupyter nbextension enable init_cell/main

COPY nbgrader-0.8.0.dev0-py2.py3-none-any.whl /tmp/nbgrader-0.8.0.dev0-py2.py3-none-any.whl
RUN pip install /tmp/nbgrader-0.8.0.dev0-py2.py3-none-any.whl
RUN jupyter nbextension install --sys-prefix --py nbgrader --overwrite
RUN jupyter nbextension enable --sys-prefix --py nbgrader
RUN jupyter serverextension enable --sys-prefix --py nbgrader
COPY nbgrader_config.py /etc/jupyter/

USER root

ENV JULIA_PATH /usr/local/julia
ENV PATH $JULIA_PATH/bin:$PATH

ENV JULIA_GPG 3673DF529D9049477F76B37566E3C7DC03D6E495

ENV JULIA_VERSION 1.8.0-rc3

RUN set -eux; \
	\
	savedAptMark="$(apt-mark showmanual)"; \
	if ! command -v gpg > /dev/null; then \
		apt-get update; \
		apt-get install -y --no-install-recommends \
			gnupg \
			dirmngr \
		; \
		rm -rf /var/lib/apt/lists/*; \
	fi; \
	\
# https://julialang.org/downloads/#julia-command-line-version
# https://julialang-s3.julialang.org/bin/checksums/julia-1.8.0-rc3.sha256
	arch="$(dpkg --print-architecture)"; \
	case "$arch" in \
		'amd64') \
			url='https://julialang-s3.julialang.org/bin/linux/x64/1.8/julia-1.8.0-rc3-linux-x86_64.tar.gz'; \
			sha256='7c57ac9aabce9b3cdc0f7bb1c393fe8058b794ddbb4168fcb28800b6459aa815'; \
			;; \
		'arm64') \
			url='https://julialang-s3.julialang.org/bin/linux/aarch64/1.8/julia-1.8.0-rc3-linux-aarch64.tar.gz'; \
			sha256='40ae51fe3d20b27b40163daef761c2565529554bca01eb38bb19532400165586'; \
			;; \
		'i386') \
			url='https://julialang-s3.julialang.org/bin/linux/x86/1.8/julia-1.8.0-rc3-linux-i686.tar.gz'; \
			sha256='047775e2f21facf288422ed7f60459c37b9083fadf24d2d6740fd965c7e940be'; \
			;; \
		*) \
			echo >&2 "error: current architecture ($arch) does not have a corresponding Julia binary release"; \
			exit 1; \
			;; \
	esac; \
	\
	curl -fL -o julia.tar.gz.asc "$url.asc"; \
	curl -fL -o julia.tar.gz "$url"; \
	\
	echo "$sha256 *julia.tar.gz" | sha256sum --strict --check -; \
	\
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$JULIA_GPG"; \
	gpg --batch --verify julia.tar.gz.asc julia.tar.gz; \
	command -v gpgconf > /dev/null && gpgconf --kill all; \
	rm -rf "$GNUPGHOME" julia.tar.gz.asc; \
	\
	mkdir "$JULIA_PATH"; \
	tar -xzf julia.tar.gz -C "$JULIA_PATH" --strip-components 1; \
	rm julia.tar.gz; \
	\
	apt-mark auto '.*' > /dev/null; \
	[ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; \
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	\
# smoke test
	julia --version

COPY packages.jl /tmp/packages.jl
RUN JULIA_DEPOT_PATH=/usr/local/julia/share/julia julia /tmp/packages.jl
COPY setup_ijulia.jl /tmp/setup_ijulia.jl
RUN JUPYTER_DATA_DIR=/opt/conda/share/jupyter julia /tmp/setup_ijulia.jl

USER ${NB_USER}
ENTRYPOINT ["tini", "--"]

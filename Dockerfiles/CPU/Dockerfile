FROM ubuntu:16.04

ENV OCPU_BRANCH 2.0.6
ENV RSTUDIO_VERSION 1.1.423

# base packages
RUN apt-get update && apt-get install -y \
	build-essential git python3-pip python-setuptools graphviz \
	pandoc curl libcurl4-openssl-dev libssl-dev ca-certificates \
	software-properties-common make cmake devscripts \
	apache2 apache2-dev libapreq2-dev libapparmor-dev apparmor-profiles \
	libprotobuf-dev protobuf-compiler xvfb language-pack-en-base \
	xvfb xauth xfonts-base libxml2-dev libxt-dev nano \
	libicu-dev pkg-config libssh2-1-dev locales apt-utils wget texinfo

RUN apt-get update && apt-get install -y \
	libgeos-dev libgdal-dev libproj-dev \
	libpng-dev libjpeg-dev libjpeg-turbo8-dev libtiff-dev fftw3-dev \
	libopenblas-dev liblapack-dev libopencv-dev

# install R 
RUN echo "deb http://cran.rstudio.com/bin/linux/ubuntu xenial/" >> /etc/apt/sources.list && \
	gpg --keyserver keyserver.ubuntu.com --recv-key E084DAB9 && \
	gpg -a --export E084DAB9 | apt-key add - && \
	apt-get update && apt-get install -y r-base r-base-dev

# jupyter
RUN python3 -m pip install pip -U \
    && python3 -m pip install jupyter

RUN R -e "install.packages(c('repr', 'IRdisplay', 'evaluate', 'crayon', 'pbdZMQ', 'devtools', 'uuid', 'digest'), repos='http://cran.rstudio.com/')" && \
	R -e "install.packages('plotly', repos='http://cran.rstudio.com/')" && \
	R -e "devtools::install_github('IRkernel/IRkernel')" && \
	R -e "IRkernel::installspec()" && \
	R -e "IRkernel::installspec(user = FALSE)"

# mxnet
RUN apt-get install -y libjemalloc-dev google-perftools

RUN git clone --recursive https://github.com/apache/incubator-mxnet && \
	cd incubator-mxnet && \
	make -j $(nproc) USE_OPENCV=1 USE_BLAS=openblas USE_MKLDNN=0

#RUN ln -s /incubator-mxnet/lib/libmklml_intel.so /lib/libmklml_intel.so && \
#	ln -s /incubator-mxnet/lib/libiomp5.so /lib/libiomp5.so && \
#	ln -s /incubator-mxnet/lib/libmklml_gnu.so /lib/libmklml_gnu.so && \
#	ln -s /incubator-mxnet/lib/libmkldnn.so /lib/libmkldnn.so && \
#	ln -s /incubator-mxnet/lib/libmkldnn.so.0 /lib/libmkldnn.so.0

RUN cd incubator-mxnet && make rpkg

# xgboost
RUN git clone --recursive https://github.com/dmlc/xgboost && \
	cd xgboost && mkdir build && cd build && \
	cmake .. -DUSE_CUDA=OFF -DR_LIB=ON && make install -j

# lightgbm
RUN git clone --recursive https://github.com/Microsoft/LightGBM && \
	cd LightGBM/R-package && \
	R CMD INSTALL --build . --no-multiarch

# rstudio
RUN apt-get install -y gdebi-core git sudo && \
	wget --quiet https://download2.rstudio.org/rstudio-server-${RSTUDIO_VERSION}-amd64.deb && \
	gdebi --non-interactive rstudio-server-${RSTUDIO_VERSION}-amd64.deb && \
	rm -f rstudio-server-${RSTUDIO_VERSION}-amd64.deb

# opencpu
RUN useradd -ms /bin/bash builder

USER builder

RUN cd ~/ && wget https://github.com/opencpu/opencpu-server/archive/v${OCPU_BRANCH}.tar.gz && \
	tar xzf v${OCPU_BRANCH}.tar.gz && \
	cd opencpu-server-${OCPU_BRANCH} && \
	dpkg-buildpackage -us -uc

USER root

RUN apt-get install -y libapache2-mod-r-base && \
	dpkg -i /home/builder/opencpu-lib_*.deb && \
	dpkg -i /home/builder/opencpu-server_*.deb

RUN ln -sf /proc/self/fd/1 /var/log/apache2/access.log && \
	ln -sf /proc/self/fd/1 /var/log/apache2/error.log && \
	ln -sf /proc/self/fd/1 /var/log/opencpu/apache_access.log && \
	ln -sf /proc/self/fd/1 /var/log/opencpu/apache_error.log

# Set opencpu password so that we can login
RUN echo "opencpu:opencpu" | chpasswd

# Custom server.conf for pre-loading specific packages in ocpu
COPY server.conf /etc/opencpu/server.conf

# R packages
RUN R -e "install.packages(c('devtools', 'plotly', 'lubridate', 'DT', 'AUC', 'formatR'), repos='http://cran.rstudio.com/')"
RUN R -e "install.packages('data.table', type = 'source', repos = 'http://Rdatatable.github.io/data.table')"
#RUN R -e "devtools::install_github('jeremiedb/PredictionEngine')"

EXPOSE 80
EXPOSE 443
EXPOSE 8004
EXPOSE 8787

# Start non-daemonized webserver
CMD /usr/lib/rstudio-server/bin/rserver && apachectl -DFOREGROUND

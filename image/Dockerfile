# Start from Jupyter image
FROM jupyter/scipy-notebook:584e9ab39d22
# Change permissions on data directory (for OpenShift).
# See https://developers.redhat.com/blog/2020/10/26/adapting-docker-and-kubernetes-containers-to-run-on-red-hat-openshift-container-platform/
USER root
RUN chgrp -R 0 /home/jovyan && \
    chmod -R 777 /home/jovyan
# Install Qiskit for the right user
USER $NB_UID
## Requirements.txt is a relevant file for the
## installation of qiskit components.
## Copying inside docker to use them later.
COPY ./requirements.txt .
## version of these is too high
Run pip uninstall -y torch numba  jupyterlab_server decorator h5py numpy scipy sympy
## install required packages
RUN pip install -U -r requirements.txt --ignore-installed llvmlite \
    && \
    pip check

#RUN pip install qiskit[visualization]

# Start from Jupyter image
FROM jupyter/scipy-notebook:584e9ab39d22
# Install Qiskit
RUN pip install qiskit[visualization]
# Change permissions on data directory
RUN chmod -R 777 /home/jovyan
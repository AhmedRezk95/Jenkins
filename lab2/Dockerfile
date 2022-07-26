FROM jenkins/jenkins:lts
USER root
# Install docker client
RUN apt-get update -qq
# Install dependencies
RUN apt-get install -qq apt-transport-https ca-certificates curl gnupg2 software-properties-common
# Add Docker’s GPG Key (remember it is debian not linux ) / don't set it as sudo !
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
# Install the Docker Repository "remember it is debian not linux)"
RUN add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/debian \
    $(lsb_release -cs) \
    stable"
# Update Repositories  &  Install Latest Version of Docker
RUN apt-get update -qq \
    && apt-get install docker-ce -y
# Add the user jenkins to the group docker on the system 
RUN usermod -aG docker jenkins

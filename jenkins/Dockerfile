FROM jenkins/jenkins:latest-jdk8
USER root
RUN apt-get update && apt-get install -y lsb-release curl net-tools wget unzip ansible awscli software-properties-common
RUN curl -fsSL https://apt.releases.hashicorp.com/gpg | apt-key add -
RUN apt-add-repository "deb [arch=$(dpkg --print-architecture)] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
 signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
 https://download.docker.com/linux/debian \
 $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
# install docker
RUN apt-get update && apt-get install -y docker-ce-cli terraform
#ensures that /var/run/docker.sock exists
RUN touch /var/run/docker.sock && chown root:jenkins /var/run/docker.sock 
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.25.2 docker-workflow:1.26"
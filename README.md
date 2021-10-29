# jenkins-paylasilan-kutuphane-ornegi
Jenkins job içinde kullanılmak üzere ortak kütüphane örneği
Bu kütüphaneyi şöyle bir dockerfile içinde çalıştırabiliriz:

```
FROM jenkins/jenkins:lts-jdk11 AS jenkins-base


#--------------------------------------------------------------------------------------------------------------------------------#
#                                         JENKINS HTTP/HTTPS AYARLARI                                                            #
#                                                                                                                                #
# Eğer jenkins MASTER'ın httpS çalışması isteniyorsa ve sertifikayı biz sağlayacaksak açık ve gizli anahtarları                  #
# /var/lib/jenkins/cert ve /var/lib/jenkins/pk dizinlerine  kopyalayarak bu anahtarların jenkins grubundaki jenkins              #
# kullanıcısına aitliğini verir:                                                                                                 #
#   COPY --chown=jenkins:jenkins https.pem /var/lib/jenkins/cert                                                                 #
#   COPY --chown=jenkins:jenkins https.key /var/lib/jenkins/pk                                                                   #
#                                                                                                                                #
# JENKINS_OPTS isimli ortam değişkenine ilgili anahtarlarla yazarız:                                                             #
#   ENV JENKINS_OPTS --httpsCertificate=/var/lib/jenkins/cert --httpsPrivateKey=/var/lib/jenkins/pk                              #
#                                                                                                                                #
# http veya httpS çalışmasın istiyorsak --httpPort=-1 veya --httpsPort=-1 parametrelerini veririz:                               #
#                                                                                                                                #
# Jenkins'in http çalışacağı portu vermek için:                                                                                  #
#   ENV JENKINS_OPTS --httpPort=8081                                                                                             #
#                                                                                                                                #
# Jenkins'in httpS çalışacağı portu vermek için                                                                                  #
#   ENV JENKINS_OPTS --httpsPort=8082                                                                                            #
#                                                                                                                                #
#   COPY --chown=jenkins:jenkins https.pem /var/lib/jenkins/cert                                                                 #
#   COPY --chown=jenkins:jenkins https.key /var/lib/jenkins/pk                                                                   #
# ENV JENKINS_OPTS --httpPort=-1 --httpsPort=8083 --httpsCertificate=/var/lib/jenkins/cert --httpsPrivateKey=/var/lib/jenkins/pk #
#                                                                                                                                #
#--------------------------------------------------------------------------------------------------------------------------------#
ENV JENKINS_OPTS --httpPort=8083

#------------------------------------------------------------------------------------------#
#                                  JENKINS EKLENTILERİ                                     #
#                                                                                          #
# Jenkins eklentilerini ister host içinden kopyalayabilir                                  #
#   COPY ./jenkins-plugins/cem /var/jenkins_home/plugins                                   #
#                                                                                          #
# ister jenkins kurulurken yüklemesini sağlayabilirsiniz                                   #
#   RUN echo "\n\                                                                          # 
#   build-timeout:latest\n\                                                                # 
#   dark-theme:latest\n\                                                                   # 
#   configuration-as-code:latest\n\                                                        # 
#   credentials-binding:latest\n\                                                          # 
#   git:latest\n\                                                                          # 
#   github-branch-source:latest\n\                                                         # 
#   gradle:latest\n\                                                                       # 
#   pam-auth:latest\n\                                                                     # 
#   pipeline-github-lib:latest\n\                                                          # 
#   pipeline-stage-view:latest\n\                                                          # 
#   ssh-slaves:latest\n\                                                                   # 
#   timestamper:latest\n\                                                                  # 
#   workflow-aggregator:latest\n\                                                          # 
#   ws-cleanup:latest\n\                                                                   # 
#   allure-jenkins-plugin:latest\n\                                                        # 
#   job-dsl:latest\n\                                                                      # 
#   nodejs:latest\n\                                                                       # 
#   docker-plugin:latest\n\                                                                # 
#   docker-workflow:latest\n\                                                              # 
#   chromedriver:latest\n\                                                                 # 
#   " > /usr/share/jenkins/ref/plugins.txt                                                 # 
#   RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt             # 
#                                                                                          #
# -----------------------------------------------------------------------------------------#

COPY ./jenkins-plugins/plugins /var/jenkins_home/plugins
VOLUME ["/var/jenkins_home/plugins"]

# -----------------------------------------------------------------------------------------#
#                               JENKINS KURULUM SİHİRBAZI                                  #
# Jenkins master varsayılan olarak kurulum ile başlatılır. Kurulum yapmadan çalışması için #
# jenkins.install.runSetupWizard=false  işaretlenir.                                       #
#                                                                                          #
# -----------------------------------------------------------------------------------------#
ENV JAVA_OPTS -Djenkins.install.runSetupWizard=false

VOLUME ["/var/jenkins_home"]


FROM jenkins-base
# COPY casc.yaml /var/jenkins_home/casc.yaml



ENV CASC_JENKINS_CONFIG /var/jenkins_home/casc.yaml
RUN echo '\n\
jenkins:\n\
  securityRealm:\n\
    local:\n\
      allowsSignup: false\n\
      users:\n\
        - id: admin\n\
          password: admin\n\
\n\
security:\n\
  globaljobdslsecurityconfiguration:\n\
    useScriptSecurity: false\n\
\n\
unclassified:\n\
  location:\n\
    url: http://localhost:8083/\n\
  globalLibraries:\n\
    libraries:\n\
      - name: "ortak-kutuphane"\n\
        retriever:\n\
          modernSCM:\n\
            scm:\n\
              git:\n\
                remote: "https://github.com/cemtopkaya/jenkins-paylasilan-kutuphane-ornegi.git"\n\
  themeManager:\n\
    disableUserThemes: true\n\
    theme: "darkSystem" # use "dark" for forcing the dark theme regardless of OS settings\n\
\n\
jobs:\n\
  - script: >\n\
      freeStyleJob("basit-freestyle-job") {\n\
        steps {\n\
            shell "echo freestyle içinden merhaba"\n\
        }\n\
      }\n\
  - script: queue("basit-freestyle-job")\n\
  - script: |\n\
        freeStyleJob("TOOL_JobsMakel").with {\n\
            displayName("Deploy Jenkins job definitions")\n\
            label("master")\n\
            parameters {\n\
                stringParam("SCM_BRANCH","master", "Source branch for SCM repository, default is master")\n\
            }\n\
            scm {\n\
                git("${SCM_REPO}", "${SCM_BRANCH}")\n\
            }\n\
            steps {\n\
                dsl(["jenkins_jobs/*.groovy"])\n\
            }\n\
        }\n\
  - script: >\n\
      pipelineJob("jenkinsfile-ornegi") {\n\
        definition {\n\
          cpsScm {\n\
            scm {\n\
              git {\n\
                remote {\n\
                  url("https://github.com/cemtopkaya/jenkinsfile-node-script-ornegi.git")\n\
                }\n\
                branch("*/main")\n\
              }\n\
            }\n\
            lightweight()\n\
          }\n\
        }\n\
      }\n\
  - script: queue("jenkinsfile-ornegi")\n\
' >  /var/jenkins_home/casc.yaml

EXPOSE 8083
```

# jenkins pipeline使用私有仓库中镜像

拉取私有仓库的镜像需要使用私有仓库的认证信息，认证信息做成secret的方式。docker daemon需要添加仓库的认证证书。 secret添加的方式：

```text
kubectl create secret docker-registry artifactory --namespace=testcenter --docker-server=docker-local.repo.inspur.com --docker-username=username  --docker-password='password' --docker-email=user email
```

pipeline中添加secret引用

```text
def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: label, containers: [
    //containerTemplate(name: 'maven', image: 'maven:3.3.9-jdk-8-alpine', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'golang', image: 'docker-local.repo.inspur.com/jenkinsci/golang:1.8.0', ttyEnabled: true, command: 'cat')
  ],
  imagePullSecrets: ["artifactory"]
  ) {

    node(label) {

        stage('Get a Golang project') {
            git url: 'https://github.com/hashicorp/terraform.git'
            container('golang') {
                stage('Build a Go project') {
                    sh """
                    mkdir -p /go/src/github.com/hashicorp
                    ln -s `pwd` /go/src/github.com/hashicorp/terraform
                    cd /go/src/github.com/hashicorp/terraform && make core-dev
                    """
                }
            }
        }

    }
}
```


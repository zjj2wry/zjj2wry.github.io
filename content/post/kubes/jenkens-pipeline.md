---
title: "Jenkens Pipeline"
date: 2019-09-25T11:27:54+08:00
lastmod: 2019-09-25T11:27:54+08:00
tags: [ "jenkins" ]
categories: [ "jenkins" ]
---

jenkins pipeline 分为 Scripted Pipeline 和 Declarative Pipeline，Scripted Pipeline 是最早基于 groovy 实现的 pipeline 语法，Declarative Pipeline 是后面推出的声明式 pipeline 语法，两个最大的区别就是 Declarative Pipeline 整体比较简洁，学习起来也比 groovy 要快的多。Declarative Pipeline 的编程模型是 [Declarative programming](https://en.wikipedia.org/wiki/Declarative_programming)，而 Scripted Pipeline 是 [Imperative_programming](https://en.wikipedia.org/wiki/Imperative_programming)。 Declarative Pipeline 要预定义脚本结构，语法也比较严格一些，Scripted Pipeline 其实就是一行一行的脚本，一般执行到那才知道有没有问题。

```
Jenkinsfile (Scripted Pipeline)
node {
    stage('Example') {
        if (env.BRANCH_NAME == 'master') {
            echo 'I only execute on the master branch'
        } else {
            echo 'I execute elsewhere'
        }
    }
}
```

```
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent { docker 'maven:3-alpine' } 
    stages {
        stage('Example Build') {
            steps {
                sh 'mvn -B clean verify'
            }
        }
    }
}
```

最终还是选择使用了 Declarative Pipeline，因为比较简洁，而且能满足所有的需求。在使用 Declarative Pipeline 可以先看一下 [官方语法文档](https://jenkins.io/doc/book/pipeline/syntax/#compare) 和 [官方例子仓库](https://github.com/jenkinsci/pipeline-examples)。

## maybe help

### git parameter

Declarative Pipeline 一样可以定义参数化构建，有一些情况你可能不想把 jenkinsfile 和代码的仓库放到一下，但是你要 checkout code，所以需要定义 gitParameter 来拉指定分支或者 tag 的代码。这个有一个问题要先执行一次才能在 jenkins 的界面上拉分支和 tag

```
  parameters {
    gitParameter defaultValue: 'origin/master',
             name: 'BRANCH_TAG',
             type: 'PT_BRANCH_TAG',
             useRepository: "<git_url>",
             quickFilterEnabled: 'true',
             listSize: '10',
             description: 'pre 和 prd 环境使用 tag 发布',
             branchFilter: 'origin.*/(.*)',
             tagFilter: '*'
  }
```
### 定义环境变量

```
  environment {
    PROJECT="${JOB_NAME}"
    DATE=sh(returnStdout: true, script: 'date "+%Y-%m-%d %H:%M:%S"').trim()
  }
```

### checkout scm

```
        container('go') {
          checkout scm: [$class: 'GitSCM', 
                   userRemoteConfigs: [[url: "<git_url>", credentialsId: 'gitlab-admin-ssh']], 
                   branches: [[name: "${params.BRANCH_TAG}"]]], 
                   poll: false
        }
```

### 获取 git 版本信息

因为发布的时候可以选择 分支和 tag，所以在参数化构建的时候定义 BRANCH_TAG 不能直接作为 tag 来使用，可以使用 git 命令来获取 tag 信息

```
        script {
          VERSION=sh(returnStdout: true, script: 'git describe --candidate=1 --tags').trim()
        }  
```

### if else
```
			script {
            if (env.BUILD_IMAGE == "false"){
              sh "docker pull ..."
            } else {
            	 sh "docker build ..."
            }
				}
```

### git 

在 pipeline 中要使用其他的 git 仓库，下面会 clone 其他的仓库到 _chart 目录。

```
          dir('_chart') {
            git branch: "<git_branch>", url: "<git_url>", credentialsId: 'gitlab-admin-ssh'
          } 
```

在使用的时候想把最后部署到 k8s 的所有 yaml 文件做一个备份，并记录一下用户名，下面的例子是怎么获取执行的用户名和如何在 credentials 中拿到 gitlab 的用户名和密码。

```
            wrap([$class: 'BuildUser']) {
              script {
                withCredentials([usernamePassword(credentialsId: 'zhengjiajin-gitlab', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                  sh("""export GIT_AUTHOR_EMAIL=******
                    export GIT_AUTHOR_NAME=*******
                    git add ${JOB_NAME}/${ENV}/${BUILD_ID}/_deploy.yaml
                    git commit -m '${BUILD_USER_ID} prompt ${JOB_NAME}:${VERSION} to ${ENV} in ${DATE}'
                    git push https://${GIT_USERNAME}:${GIT_PASSWORD}@<git_url>""")
                }
              }
            }
```

### post

发布的时候可能想有一个通知，可以使用 post 语句, `git log --pretty=format:"%h - %an, %ar : %s" -n 1'` 可以显示当前 git 最后一个 commit 信息。

```
  post {
    always {
        container('go') {
            wrap([$class: 'BuildUser']) {
                script {
                  text=sh(returnStdout: true, script: 'git log --pretty=format:"%h - %an, %ar : %s" -n 1').trim()
                  endAt=sh(returnStdout: true, script: 'date "+%Y-%m-%d %H:%M:%S"').trim()
                  end=sh(returnStdout: true, script: 'date "+%s"').trim()
                  result=currentBuild.result
                  duration= (end as int)-(start as int)
                }
                sh """curl -i -X POST -H "Content-type: application/json" -d '
{
  "operator": "${BUILD_USER_ID}",
  "serviceName":"${JOB_NAME}",
  "startAt":"${startAt}",
  "endAt":"${endAt}",
  "text": "${text}",
  "duration":"${duration}s",
  "jobAddress": "${JOB_URL}",
  "status":"${result}"
}' ${url}"""
              }
        }
    }
  }
```



### load

Declarative Pipeline 好像并不支持 load 其他 pipeline 的能力，Scripted Pipeline 是可以 load 其他的 pipeline 的，在实际使用的时候遇到一个问题就是 k8s 的 yaml 和 jenkins 都需要一些学习曲线，用户完全不关心，这个时候可以把 pipeline 放到另一个仓库中作为 code 来管理，然后使用 load 加载 pipeline。下面的例子是通过 gitlab raw 的 API 获取 Declarative Pipelines 并加载执行，另外把想要在 jenkins 上修改的变量 echo 到了 _Jenkinsfile 顶部，比较 hack。

```
node("jenkins-go"){
  stage("load declarative jenkinsfile"){
    container('go') {
      withCredentials([string(credentialsId: 'gitlab-token', variable: 'TOKEN')]) {
        sh """
          set +vx
          echo "def GIT_URL = '<GIT_URL>'
// 指定部署文件的仓库(jenkinsfile 和 k8s yaml 文件)
def CHART_GIT_BRANCH = 'master'
def CHART_GIT_URL = '<git_url>'" > _Jenkinsfile
          curl -H "PRIVATE-TOKEN: ${TOKEN}" "https://<git_url>/api/v4/projects/6082/repository/files/jobs%2F${JOB_NAME}%2FJenkinsfile/raw?ref=master">>_Jenkinsfile
        """
      }
    }
  }
  load "_Jenkinsfile"
}
```

需要完整的例子可以发邮件给我。
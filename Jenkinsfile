pipeline {
    agent {
        node {
            label 'metersphere'
        }
    }
    options { 
        quietPeriod(600)
        checkoutToSubdirectory('installer')
    }
    parameters { 
        string(name: 'IMAGE_PREFIX', defaultValue: 'registry.cn-qingdao.aliyuncs.com/metersphere', description: '构建后的 Docker 镜像带仓库名的前缀')
    }
    environment {
        BRANCH = 'v1.6'
        RELEASE = 'v1.6.0-rc1'
        IMAGE_PREFIX = "${params.IMAGE_PREFIX}"
    }
    stages {
        stage('Preparation') {
            when { tag "v*" }
            steps {
                // Get some code from a GitHub repository
                dir('ms-server') {
                    git credentialsId:'metersphere-registry', url: 'git@github.com:metersphere/metersphere.git', branch: "${BRANCH}"
                }
                dir('ms-node-controller') {
                    git credentialsId:'metersphere-registry', url: 'git@github.com:metersphere/node-controller.git', branch: "${BRANCH}"
                }
                dir('ms-data-streaming') {
                    git credentialsId:'metersphere-registry', url: 'git@github.com:metersphere/data-streaming.git', branch: "${BRANCH}"
                }
                sh '''
                    git config --global user.email "wangzhen@fit2cloud.com"
                    git config --global user.name "BugKing"
                '''
            }
        }
        stage('Tag Other Repos') {
            when { tag "v*" }
            parallel {
                stage('ms-server') {
                    steps {
                        dir('ms-server') {
                            sh("git tag -f -a ${RELEASE} -m 'Tagged by Jenkins'")
                            sh("git push -f origin refs/tags/${RELEASE}")
                        }
                        build job:"../metersphere/${RELEASE}", quietPeriod:10
                    }
                }
                stage('ms-node-controller') {
                    steps {
                        dir('ms-node-controller') {
                            sh("git tag -f -a ${RELEASE} -m 'Tagged by Jenkins'")
                            sh("git push -f origin refs/tags/${RELEASE}")
                        }
                        build job:"../node-controller/${RELEASE}", quietPeriod:10
                    }
                }
                stage('ms-data-streaming') {
                    steps {
                        dir('ms-data-streaming') {
                            sh("git tag -f -a ${RELEASE} -m 'Tagged by Jenkins'")
                            sh("git push -f origin refs/tags/${RELEASE}")
                        }
                        build job:"../data-streaming/${RELEASE}", quietPeriod:10
                    }
                }
            }
        }   
        stage('Package') {
            steps {
                dir('installer') {
                    script {
                        def images = ['jmeter-master:5.3-ms14',
                                    'kafka:2',
                                    'zookeeper:3',
                                    'mysql:5.7.25',
                                    'metersphere:${RELEASE}',
                                    "ms-node-controller:${RELEASE}",
                                    "ms-data-streaming:${RELEASE}"]
                        for (image in images) {
                            waitUntil {
                                def r = sh script: "docker pull ${IMAGE_PREFIX}/${image}", returnStatus: true
                                r == 0;
                            }
                        }
                    }
                    sh '''
                        echo ${RELEASE}-b$BUILD_NUMBER > ./metersphere/version
                        #保存镜像
                        rm -rf images && mkdir images && cd images
                        docker save ${IMAGE_PREFIX}/metersphere:${RELEASE} -o metersphere.tar
                        docker save ${IMAGE_PREFIX}/ms-node-controller:${RELEASE} -o ms-node-controller.tar
                        docker save ${IMAGE_PREFIX}/ms-data-streaming:${RELEASE} -o ms-data-streaming.tar
                        docker save ${IMAGE_PREFIX}/jmeter-master:5.3-ms14 -o jmeter-master.tar
                        docker save ${IMAGE_PREFIX}/kafka:2 -o kafka.tar
                        docker save ${IMAGE_PREFIX}/zookeeper:3 -o zookeeper.tar
                        docker save ${IMAGE_PREFIX}/mysql:5.7.25 -o mysql.tar
                        cd ..

                        #修改安装参数
                        sed -i -e "s#MS_TAG=.*#MS_TAG=${RELEASE}#g" install.conf
                        sed -i -e "s#MS_PREFIX=.*#MS_PREFIX=${IMAGE_PREFIX}#g" install.conf
                        sed -i -e "s#MS_JMETER_TAG=.*#MS_JMETER_TAG=5.3-ms14#g" install.conf

                        #获取docker
                        rm -rf docker*
                        wget http://fit2cloud2-offline-installer.oss-cn-beijing.aliyuncs.com/tools/docker.zip
                        unzip docker.zip
                        rm -rf __MACOSX
                        rm -rf docker.zip

                        rm -rf metersphere-release*.tar.gz

                        #打包离线包
                        touch metersphere-release-${RELEASE}-offline.tar.gz
                        tar czvf metersphere-release-${RELEASE}-offline.tar.gz . --transform "s/^\\./metersphere-release-${RELEASE}-offline/" \\
                            --exclude metersphere-release-${RELEASE}-offline.tar.gz \\
                            --exclude .git

                        #打包在线包
                        touch metersphere-release-${RELEASE}.tar.gz
                        tar czvf metersphere-release-${RELEASE}.tar.gz . --transform "s/^\\./metersphere-release-${RELEASE}/" \\
                            --exclude metersphere-release-${RELEASE}.tar.gz \\
                            --exclude metersphere-release-${RELEASE}-offline.tar.gz \\
                            --exclude .git \\
                            --exclude images \\
                            --exclude docker
                    '''
                }
            }
        }

        stage('Release') {
            when { tag "v*" }
            steps {
                withCredentials([string(credentialsId: 'gitrelease', variable: 'TOKEN')]) {
                    withEnv(["TOKEN=$TOKEN", "branch=$BRANCH", "RELEASE=$RELEASE"]) {
                        dir('installer') {
                            sh script: '''
                                release=$(curl -XPOST -H "Authorization:token $TOKEN" --data "{\\"tag_name\\": \\"${RELEASE}\\", \\"target_commitish\\": \\"${branch}\\", \\"name\\": \\"${RELEASE}\\", \\"body\\": \\"\\", \\"draft\\": false, \\"prerelease\\": true}" https://api.github.com/repos/metersphere/metersphere/releases)
                                id=$(echo "$release" | sed -n -e \'s/"id":\\ \\([0-9]\\+\\),/\\1/p\' | head -n 1 | sed \'s/[[:blank:]]//g\')
                                curl -XPOST -H "Authorization:token $TOKEN" -H "Content-Type:application/octet-stream" --data-binary @quick_start.sh https://uploads.github.com/repos/metersphere/metersphere/releases/${id}/assets?name=quick_start.sh
                                curl -XPOST -H "Authorization:token $TOKEN" -H "Content-Type:application/octet-stream" --data-binary @metersphere-release-${RELEASE}.tar.gz https://uploads.github.com/repos/metersphere/metersphere/releases/${id}/assets?name=metersphere-release-${RELEASE}.tar.gz
                            '''
                        }
                    }
                }
            }
        }
        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'installer/*.tar.gz,installer/quick_start.sh,installer/*.md5', followSymlinks: false
            }
        }
    }
    post('Notification') {
        always {
            sh "echo \$WEBHOOK\n"
            withCredentials([string(credentialsId: 'wechat-bot-webhook', variable: 'WEBHOOK')]) {
                qyWechatNotification failSend: true, mentionedId: '', mentionedMobile: '', webhookUrl: "$WEBHOOK"
            }
        }
    }
}
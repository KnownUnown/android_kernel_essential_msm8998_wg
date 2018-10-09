// -*-Groovy-*-

pipeline {
    agent {
        node {
            label 'linux-builder'
        }
    }

    environment {
        CC = 'clang'
        ARCH = 'arm64'
        CROSS_COMPILE = "${pwd()}/toolchain/ndk/toolchains/aarch64-linux-android-4.9/prebuilt/linux-x86_64/bin/aarch64-linux-android-"
        PATH = "${pwd()}/toolchain/clang/bin:${env.PATH}"
        LD_LIBRARY_PATH = "${pwd()}/toolchain/clang/lib64:${env.LD_LIBRARY_PATH}"
        CLANG_TRIPLE = 'aarch64-linux-gnu-'
        GIT_TAG = sh('returnStdout': true, script: 'git describe --abbrev=0 --tags').trim()
        GITHUB_RELEASE_KEY = credentials('jenkins-gh-release-key')
    }

    stages {
        stage('Build: Preparation') {
            steps {
                checkout scm: [$class: 'RepoScm', manifestRepositoryUrl: pwd(), manifestBranch: "refs/tags/${env.GIT_TAG}"]
                dir('toolchain') {
                    dir('clang') {
                        sh 'curl https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/master/clang-4053586.tar.gz > clang.tar.gz'
                        sh 'tar -xzf clang.tar.gz && rm clang.tar.gz'
                    }
                    dir('ndk') {
                        sh 'curl https://dl.google.com/android/repository/android-ndk-r16b-linux-x86_64.zip > ndk.zip'
                        sh 'unzip -q ndk.zip && rm ndk.zip'
                        sh 'mv android-ndk-r16b/* . && rm -r android-ndk-r16b'
                    }

                }
            }
        }

        stage('Build: Kernel') {
            steps {
                dir('kernel') {
                    sh 'CC=clang make mata_defconfig'
                    sh 'CC=clang make'
                }
            }
        }

        /*
        stage('Build: WLAN Module') {
            steps {
                dir('qcacld-3.0') {
                    sh ''
                    sh 'CC=clang make'
                    sh '${CROSS_COMPILE}strip --strip-debug wlan.ko'
                }
            }
        }
        */

        stage('Package') {
            steps {
                dir("AnyKernel2") {
                    checkout(['$class': 'GitSCM', branches: [[ $name: '9d7f85807fd32f44dda8eaf89ec0bab9977b82cd' ]],
                              userRemoteConfigs: [[ url: 'https://github.com/osm0sis/AnyKernel2' ]]])

                    sh 'mv -f ../kernel/arch/arm64/boot/Image.gz-dtb zImage'
                    sh 'mv -f ../anykernel.sh .'
                    sh 'sed -i "" "s/{TAG}/$GIT_TAG/g" anykernel.sh'
                    sh 'zip -r9 ../kernel.$GIT_TAG.zip * -x .git README.md *placeholder'
                }
                archiveArtifacts artifacts: 'kernel*.zip'
            }
        }

        stage('Upload') {
            steps {
                sh './upload-github-release-asset.sh github_api_token=$GITHUB_RELEASE_KEY owner=KnownUnown repo=android_kernel_essential_msm8998_wg tag=$GIT_TAG filename=./kernel.$GIT_TAG.zip'
            }
        }
    }

    post {
        cleanup {
            cleanWs cleanWhenAborted: true, cleanWhenFailure: false
        }
    }
}

pipeline {
    agent none

    options {
        /* The artifacts are reasonably large, so don't keep too much history around. */
        buildDiscarder(logRotator(numToKeepStr: '1'))
        timestamps()
    }

    parameters {
        booleanParam(name: 'Linux_Build', defaultValue: true, description: 'Build the shared libraries on Linux')
        booleanParam(name: 'Windows_Build', defaultValue: true, description: 'Build the shared libraries on Windows')
    }

    environment {
        /* Only the libraries are of interest here, so skip everything that just slows the build down.
         * The build type is appended to the library names so the Release and Debug artifacts don't clash. */
        CMAKE_COMMON_OPTIONS = '-DBUILD_SHARED_LIBS=ON ' +
                               '-DJKQtPlotter_BUILD_EXAMPLES=OFF ' +
                               '-DJKQtPlotter_BUILD_FORCE_NO_PRINTER_SUPPORT=ON ' +
                               '-DJKQtPlotter_BUILD_INCLUDE_FIRAMATH_FONTS=OFF ' +
                               '-DJKQtPlotter_BUILD_INCLUDE_XITS_FONTS=OFF ' +
                               '-DJKQtPlotter_BUILD_LIB_JKQTFASTPLOTTER=OFF ' +
                               '-DJKQtPlotter_BUILD_TOOLS=OFF ' +
                               '-DJKQtPlotter_BUILD_TESTS=OFF ' +
                               '-DJKQtPlotter_BUILD_DECORATE_LIBNAMES_WITH_BUILDTYPE=ON'
        /* Every build produces both configurations. */
        BUILD_TYPES = 'Release,Debug'
    }

    stages {
        stage('Build Libraries') {
            parallel {
                stage('Linux') {
                    when { expression { params.Linux_Build } }
                    agent {
                        docker  {
                            image 'docker-registry.ampcontrol.com.au/gip/pip_app_build_image:r3.4' // The Docker image to be pulled and used for this stage
                            label 'docker_linux'          // Ensure that the agent is running Linux
                            registryUrl 'https://docker-registry.ampcontrol.com.au/'
                        }
                    }
                    steps {
                        script {
                            /* Make-style generators are single-config, so each build type needs its own build tree. */
                            for (buildType in env.BUILD_TYPES.split(',')) {
                                echo "Building JKQTPlotter shared libraries for Linux (${buildType}).."

                                /* Configure, build and install into a staging directory so the shared objects
                                 * (including the versioned symlink targets) end up in one predictable place. */
                                sh """
                                    # start the SDK environment before running cmake.. These need to be done in the same shell session so the environment variables stay in scope
                                    . /opt/PIP_OS/environment-setup-core2-64-poky-linux
                                    rm -rf build_linux_${buildType} dist_linux/${buildType}
                                    cmake -S . -B build_linux_${buildType} \
                                          -DCMAKE_BUILD_TYPE=${buildType} \
                                          -DCMAKE_INSTALL_PREFIX=\$(pwd)/dist_linux/${buildType} \
                                          ${env.CMAKE_COMMON_OPTIONS}
                                    cmake --build build_linux_${buildType} --parallel \$(nproc)
                                    cmake --install build_linux_${buildType}
                                """
                            }
                        }

                        /* .so (and the versioned .so.X.Y.Z files) live in the install lib directory. */
                        archiveArtifacts artifacts: 'dist_linux/*/lib*/**/*.so*', fingerprint: true
                    }
                    post {
                        failure {
                            deleteDir() /* clean up our workspace */
                        }
                    }
                }

                stage('Windows') {
                    when { expression { params.Windows_Build } }
                    agent { label 'Win' }
                    steps {
                        /* Start from a clean tree, the Windows nodes are not containerised. */
                        bat 'git clean -q -d -f -x'

                        /* Visual Studio is a multi-config generator, so one configure step covers both build types. */
                        bat """
                            cmake -S . -B build_windows -A x64 ^
                                  ${env.CMAKE_COMMON_OPTIONS}
                        """

                        script {
                            for (buildType in env.BUILD_TYPES.split(',')) {
                                echo "Building JKQTPlotter shared libraries for Windows (${buildType}).."
                                bat "cmake --build build_windows --config ${buildType} --parallel"
                                bat "cmake --install build_windows --config ${buildType} --prefix %WORKSPACE%\\dist_windows\\${buildType}"
                            }
                        }

                        /* The DLLs are installed into bin/, the matching import libraries into lib/. */
                        archiveArtifacts artifacts: 'dist_windows/*/bin/**/*.dll, dist_windows/*/lib/**/*.lib', fingerprint: true
                    }
                    post {
                        failure {
                            deleteDir() /* clean up our workspace */
                        }
                    }
                }
            }
        }
    }

    // On failure, send an email to everyone who has committed since the last success
    post {
        always {
            script {
                if (currentBuild.currentResult == 'FAILURE' || currentBuild.currentResult == 'UNSTABLE') {
                    emailext subject: '$DEFAULT_SUBJECT',
                        body: '$DEFAULT_CONTENT',
                        recipientProviders: [
                            [$class: 'CulpritsRecipientProvider']
                        ],
                        replyTo: '$DEFAULT_REPLYTO',
                        to: '$DEFAULT_RECIPIENTS'
                }
            }
        }
    }
}

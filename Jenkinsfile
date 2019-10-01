pipeline
{

    agent 
    {
        node 
        {
            label "maven"
        }
    }

    parameters 
    {
        string(name: 'PROJECT_NAME', defaultValue: 'pipeline-s2i' ,description: 'What is the project name?')
        string(name: 'GIT', defaultValue: 'https://github.com/lhsribas/spring-boot-rest-example.git' ,description: 'What is the git repo?')
        string(name: 'GIT_BRANCH', defaultValue: 'master', description: 'What is the git branch name?')
    }

    environment 
    {
        def version = ""
        def artifactId = ""
        def rollout = true
    }

    stages 
    {
        stage("Checkout") 
        {
            steps 
            {
                script 
                {
                    git branch: "${params.GIT_BRANCH}", url: "${params.GIT}" 
                
                    version = readMavenPom().getVersion();
                    echo "Version ::: ${version}"

                    artifactId = readMavenPom().getArtifactId()
                    echo "artifactId ::: ${artifactId}"
                
                }
            }
        }

        stage("Test Project") 
        {
            steps 
            {
                script 
                {
                    withMaven(mavenSettingsConfig: "maven-settings") {
                        sh "mvn clean test"
                    }
                }
            }
        }

        stage("Build Project") 
        {
            steps 
            {
                script 
                {
                    withMaven(mavenSettingsConfig: "maven-settings") {
                        sh "mvn clean package"
                    }
                }
            }
        }

        stage('Create Image Builder') 
        {
            steps 
            {
                script 
                {
                    openshift.withCluster() 
                    {
                        openshift.withProject("${params.PROJECT_NAME}") 
                        {
                            echo "Using project: ${openshift.project()}"
                            if (!openshift.selector("bc", "${artifactId}").exists()) 
                            {
                                openshift.newBuild("--name=${artifactId}", "--image-stream=openshift/openjdk18-openshift:latest", "--binary")
                            }
                        }
                     }
                }
            }
        }
        
        stage('Start Build Image') 
        {
            steps 
            {
                script 
                {
                    openshift.withCluster() 
                    {
                        openshift.withProject("${params.PROJECT_NAME}") 
                        {
                            echo "Using project: ${openshift.project()}"
                            openshift.selector("bc", "${artifactId}").startBuild("--from-file=target/${artifactId}-${version}.jar", "--wait=true")
                        }
                    }
                }
            }
        }

        stage('Promote to Image') {
            steps 
            {
                script 
                {
                    openshift.withCluster() 
                    {
                        openshift.withProject("${params.PROJECT_NAME}") 
                        {
                            openshift.tag("${artifactId}:latest", "${artifactId}:${version}")
                        }
                    }
                }
            }
        }

        stage('Create ServiceAccount') 
        {
            steps 
            {
                script 
                {
                    openshift.withCluster() 
                    {
                        openshift.withProject("${params.PROJECT_NAME}") 
                        {
                            if (!openshift.selector('sa', "${artifactId}").exists()) 
                            {
                                openshift.create('sa', "${artifactId}")
                                openshift.raw('policy', 'add-role-to-user', 'view', "system:serviceaccount:${params.PROJECT_NAME}:${artifactId}")
                            }
                            
                        }
                    }
                }
            }
        }

        stage('Create DEV') {
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject("${params.PROJECT_NAME}") {
                            return !openshift.selector('dc', "${artifactId}").exists()
                        }
                    }
                }
            }

            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${params.PROJECT_NAME}") {
                            if (!openshift.selector('dc', "${artifactId}").exists()) {
                                openshift.newApp("s2i-vv-spring-boot",
                                " -p APP_NAME=${artifactId}",
                                " -p APP_VERSION=${version}",
                                " -p NAMESPACE=${params.PROJECT_NAME}",
                                " -p CPU_REQUEST=0.2",
                                " -p MEMORY_REQUEST=256Mi",
                                " -p CPU_LIMIT=1.0",
                                " -p MEMORY_LIMIT=256Mi")

                                rollout = false
                            }
                        }
                    }
                }
            }
        }

        stage('Rollout Version Container')
        {
            when {
                expression {
                    return rollout
                }
            }

            steps 
            {
                script 
                {
                    openshift.withCluster() 
                    {
                        openshift.withProject("${params.PROJECT_NAME}") 
                        {
                            echo "Using project ::: ${openshift.project()}"
                            // Getting the deploymentConfig
                            def deploymentConfig = openshift.selector("dc", "${artifactId}").object()

                            for(int a=0; a<deploymentConfig.spec.triggers.size(); a++ ) 
                            {
                                if(deploymentConfig.spec.triggers[a].toString().contains("imageChangeParams")) 
                                {
                                    if("${deploymentConfig.spec.triggers[a].imageChangeParams.from.name}" != "${artifactId}:${version}") 
                                    {
                                        echo "ContainerImage changed to ::: ${deploymentConfig.spec.triggers[a].imageChangeParams.from.name}"
                                        deploymentConfig.spec.triggers[a].imageChangeParams.from.name="${artifactId}:${version}"
                                        openshift.apply(deploymentConfig)
                                    }
                                    else
                                    {
                                        echo "Wasn't possible change the Image, because is the same to the previous."
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
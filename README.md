# Jenkins CI/CD QA Pipeline

![Jenkins](https://img.shields.io/badge/Jenkins-2.4+-D24939?style=flat-square&logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Container-2496ED?style=flat-square&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-K8s-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![Groovy](https://img.shields.io/badge/Groovy-Pipeline-4298B8?style=flat-square&logo=apachegroovy&logoColor=white)
![Shell](https://img.shields.io/badge/Shell-Bash-121011?style=flat-square&logo=gnu-bash&logoColor=white)

Jenkins CI/CD pipeline templates for **enterprise QA automation**. Features parallel regression execution across **9+ clusters**, intelligent test suite selection, automated log-tailing, and Docker/Kubernetes integration. These pipelines reduced release regression time by **83%** (30 days → 5 days) at DataCore Software.

---

## Key Achievements

- **83% reduction** in release regression time (30 days → 5 days)
- - **9+ parallel clusters** executing regression simultaneously
  - - **11 releases** delivered (3 major, 8 minor) — zero missed deadlines
    - - **Zero manual defect triage** via automated log-tailing
      - - **90% reduction** in manual QA intervention at Publicis Sapient
       
        - ---

        ## Pipeline Catalogue

        | Pipeline | Purpose | Trigger |
        |---------|---------|---------|
        | `swarm-regression` | Full Swarm regression across 9 clusters | Release tag |
        | `swarm-smoke` | Quick smoke validation | Every commit |
        | `gateway-regression` | Swarm Gateway test suite | Daily |
        | `appliance-regression` | Appliance product tests | Release tag |
        | `performance-benchmark` | WARP performance benchmarks | On-demand |
        | `ecommerce-regression` | E-commerce UI + API regression | Every commit |
        | `mobile-regression` | iOS + Android parallel execution | Daily |

        ---

        ## Architecture

        ```
        jenkins-cicd-qa-pipeline/
        ├── pipelines/
        │   ├── swarm-regression/
        │   │   ├── Jenkinsfile                    # Main Swarm regression pipeline
        │   │   └── cluster-matrix.yaml            # 9-cluster test matrix
        │   ├── swarm-smoke/
        │   │   └── Jenkinsfile
        │   ├── ecommerce-regression/
        │   │   └── Jenkinsfile
        │   └── performance-benchmark/
        │       └── Jenkinsfile
        ├── shared-library/
        │   ├── vars/
        │   │   ├── runTestSuite.groovy            # Reusable test runner
        │   │   ├── tailLogs.groovy                # Log tailing utility
        │   │   ├── notifySlack.groovy             # Slack notifications
        │   │   └── publishReport.groovy           # HTML report publisher
        │   └── src/com/qalib/
        │       ├── ClusterManager.groovy          # Cluster config manager
        │       └── TestSelector.groovy            # Intelligent test selector
        ├── docker/
        │   ├── qa-runner/
        │   │   ├── Dockerfile                     # QA test runner image
        │   │   └── entrypoint.sh
        │   └── docker-compose.yml
        ├── kubernetes/
        │   ├── qa-runner-deployment.yaml
        │   └── qa-runner-service.yaml
        └── scripts/
            ├── setup-env.sh
            ├── tail-logs.sh                        # Automated log tailing
            └── generate-report.sh
        ```

        ---

        ## Core Pipelines

        ### Swarm Parallel Regression Pipeline (`pipelines/swarm-regression/Jenkinsfile`)

        ```groovy
        // Multi-cluster parallel Swarm regression pipeline
        // Executes regression across 9 clusters simultaneously
        // Achieved 83% regression time reduction: 30 days -> 5 days

        @Library('qa-shared-library') _

        pipeline {
            agent none

            parameters {
                choice(name: 'RELEASE_VERSION', choices: ['18.0', '17.5', '17.0'],
                       description: 'Swarm release version to test')
                choice(name: 'SUITE', choices: ['full', 'smoke', 'api', 'performance'],
                       description: 'Test suite to execute')
                booleanParam(name: 'RUN_WARP_BENCHMARK', defaultValue: false,
                             description: 'Run WARP performance benchmarks')
            }

            environment {
                SLACK_CHANNEL = '#qa-alerts'
                REPORT_DIR = 'reports'
                JIRA_PROJECT = 'SWAR'
            }

            stages {
                stage('Pre-flight Checks') {
                    agent { label 'qa-master' }
                    steps {
                        script {
                            echo "Starting Swarm ${params.RELEASE_VERSION} regression"
                            sh 'python3 scripts/cluster_health_check.py --all-clusters'
                            notifySlack(
                                channel: env.SLACK_CHANNEL,
                                message: "Swarm ${params.RELEASE_VERSION} regression started",
                                color: 'good'
                            )
                        }
                    }
                }

                stage('Parallel Cluster Regression') {
                    parallel {
                        stage('Cluster: Ragnarok (EC)') {
                            agent { label 'qa-runner-ragnarok' }
                            steps {
                                runTestSuite(
                                    cluster: 'ragnarok',
                                    replication: 'EC',
                                    suite: params.SUITE,
                                    version: params.RELEASE_VERSION
                                )
                            }
                            post {
                                always {
                                    tailLogs(cluster: 'ragnarok', version: params.RELEASE_VERSION)
                                    publishReport(name: 'Ragnarok EC', path: 'reports/ragnarok')
                                }
                            }
                        }

                        stage('Cluster: Fenrir (EC)') {
                            agent { label 'qa-runner-fenrir' }
                            steps {
                                runTestSuite(
                                    cluster: 'fenrir',
                                    replication: 'EC',
                                    suite: params.SUITE,
                                    version: params.RELEASE_VERSION
                                )
                            }
                            post {
                                always {
                                    tailLogs(cluster: 'fenrir', version: params.RELEASE_VERSION)
                                }
                            }
                        }

                        stage('Cluster: Odin (Full Replica)') {
                            agent { label 'qa-runner-odin' }
                            steps {
                                runTestSuite(
                                    cluster: 'odin',
                                    replication: 'FULL_REPLICA',
                                    suite: params.SUITE,
                                    version: params.RELEASE_VERSION
                                )
                            }
                            post {
                                always {
                                    tailLogs(cluster: 'odin', version: params.RELEASE_VERSION)
                                }
                            }
                        }

                        stage('Gateway Tests') {
                            agent { label 'qa-runner-gateway' }
                            steps {
                                sh """
                                    pytest tests/gateway/ -v \
                                      --cluster=gateway \
                                      --version=${params.RELEASE_VERSION} \
                                      --html=reports/gateway/report.html
                                """
                            }
                        }

                        stage('Veeam Integration') {
                            agent { label 'qa-runner-veeam' }
                            steps {
                                sh """
                                    pytest tests/backup/veeam/ -v \
                                      --veeam-version=12 \
                                      --html=reports/veeam/report.html
                                """
                            }
                        }

                        stage('Commvault Integration') {
                            agent { label 'qa-runner-commvault' }
                            steps {
                                sh """
                                    pytest tests/backup/commvault/ -v \
                                      --cv-version=11.42 \
                                      --html=reports/commvault/report.html
                                """
                            }
                        }
                    }
                }

                stage('WARP Performance Benchmark') {
                    when { expression { params.RUN_WARP_BENCHMARK } }
                    agent { label 'qa-runner-ragnarok' }
                    steps {
                        sh """
                            pytest tests/performance/ -v \
                              --cluster=ragnarok \
                              --benchmark \
                              --html=reports/benchmark/report.html
                        """
                    }
                }

                stage('Aggregate Results') {
                    agent { label 'qa-master' }
                    steps {
                        script {
                            def results = collectResults()
                            generateAggregateReport(results)
                            if (results.failCount > 0) {
                                createJiraTickets(results.failures, env.JIRA_PROJECT)
                            }
                        }
                    }
                }
            }

            post {
                success {
                    notifySlack(
                        channel: env.SLACK_CHANNEL,
                        message: "Swarm ${params.RELEASE_VERSION} PASSED — All clusters green",
                        color: 'good'
                    )
                }
                failure {
                    notifySlack(
                        channel: env.SLACK_CHANNEL,
                        message: "Swarm ${params.RELEASE_VERSION} FAILED — Check reports",
                        color: 'danger'
                    )
                }
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'reports',
                        reportFiles: '**/report.html',
                        reportName: 'QA Regression Report'
                    ])
                }
            }
        }
        ```

        ### Shared Library: `vars/tailLogs.groovy`

        ```groovy
        // Automated log-tailing function — eliminated manual defect triage
        // Parses Elasticsearch logs for ERROR/WARN patterns and attaches to build

        def call(Map config = [:]) {
            String cluster = config.cluster ?: 'unknown'
            String version = config.version ?: 'latest'

            sh """
                python3 scripts/tail_logs.py \
                  --cluster=${cluster} \
                  --version=${version} \
                  --output=logs/${cluster}_errors.txt \
                  --patterns="ERROR,CRITICAL,PANIC,SWAR-" \
                  --since=2h
            """

            archiveArtifacts(artifacts: "logs/${cluster}_errors.txt", allowEmptyArchive: true)

            def errorCount = sh(
                script: "wc -l < logs/${cluster}_errors.txt || echo 0",
                returnStdout: true
            ).trim().toInteger()

            if (errorCount > 0) {
                echo "Found ${errorCount} error lines in ${cluster} logs"
                currentBuild.description = "${cluster}: ${errorCount} log errors"
            }
        }
        ```

        ### Docker: QA Runner Image (`docker/qa-runner/Dockerfile`)

        ```dockerfile
        FROM python:3.11-slim

        LABEL maintainer="anuj180993@gmail.com"
        LABEL description="QA Test Runner for S3/Swarm automation"

        # Install system dependencies
        RUN apt-get update && apt-get install -y \
            openjdk-17-jdk \
            curl \
            wget \
            git \
            && rm -rf /var/lib/apt/lists/*

        # Install Python test dependencies
        COPY requirements.txt /app/requirements.txt
        RUN pip install --no-cache-dir -r /app/requirements.txt

        # Install Maven
        RUN wget https://archive.apache.org/dist/maven/maven-3/3.9.0/binaries/apache-maven-3.9.0-bin.tar.gz \
            && tar -xzf apache-maven-3.9.0-bin.tar.gz -C /opt \
            && rm apache-maven-3.9.0-bin.tar.gz
        ENV PATH="/opt/apache-maven-3.9.0/bin:${PATH}"

        WORKDIR /workspace

        ENTRYPOINT ["./docker/qa-runner/entrypoint.sh"]
        ```

        ---

        ## Metrics: Before vs After

        | Metric | Before CI/CD | After CI/CD |
        |--------|-------------|------------|
        | Regression Duration | 30 days | 5 days (83% faster) |
        | Manual QA Effort | ~100% | ~40% (60% automated) |
        | Defect Triage Time | 4-6 hours/cycle | 0 (automated log-tail) |
        | Cluster Coverage | 2 clusters | 9+ clusters |
        | Parallel Jobs | 0 | 6+ simultaneous |
        | CI Trigger | Manual | Every commit |

        ---

        ## Author

        **Anuj Gupta** — Lead QA Automation Engineer / Staff SDET
        [LinkedIn](https://linkedin.com/in/anuj-gupta-sdet) | anuj180993@gmail.com

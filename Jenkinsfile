pipeline {
    agent any

    environment {
        // Git
        GIT_REPO = 'https://github.com/SoujanyaKN/odoo-db-migration.git'
        BRANCH   = 'main'

        // DB creds
        DB_USER     = 'odoo'
        DB_PASSWORD = 'odoo'

        // Odoo 17
        ODOO17_DB_CONTAINER = 'odoo17-db'
        ODOO17_DB           = 'odoo17_db'
        ODOO17_DUMP         = 'odoo17.dump'

        // Odoo 18
        ODOO18_DB_CONTAINER = 'odoo18-db'
        ODOO18_DB           = 'odoo18_db'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        /* =======================
           TOTAL ENV CLEANUP
        ======================= */
        stage('Total Environment Cleanup 🧨') {
            steps {
                sh '''
                echo "Stopping all Odoo containers..."
                docker rm -f odoo17-web odoo17-db odoo18-web odoo18-db || true

                echo "Removing unused images, volumes, networks..."
                docker system prune -af --volumes || true

                rm -f ${ODOO17_DUMP}
                df -h
                '''
            }
        }

        stage('Checkout') {
            steps {
                cleanWs()
                git branch: "${BRANCH}", url: "${GIT_REPO}"
            }
        }

        stage('Verify Docker') {
            steps {
                sh '''
                docker --version
                docker compose version || docker-compose --version
                '''
            }
        }

        stage('Start Odoo 17') {
            steps {
                sh 'docker compose -f docker/docker-compose-odoo17.yml up -d'
            }
        }

        stage('Wait for Odoo 17 DB') {
            steps {
                sh '''
                until docker exec ${ODOO17_DB_CONTAINER} pg_isready -U ${DB_USER}; do
                    sleep 5
                done
                '''
            }
        }

        stage('Dump Odoo 17 DB') {
            steps {
                sh '''
                echo "Dumping Odoo 17 database..."
                docker exec ${ODOO17_DB_CONTAINER} pg_dump \
                    -U ${DB_USER} \
                    -Fc ${ODOO17_DB} > ${ODOO17_DUMP}

                ls -lh ${ODOO17_DUMP}
                '''
            }
        }

        /* =======================
           MID PIPELINE CLEANUP
        ======================= */
        stage('Mid-Pipeline Disk Cleanup 🧹') {
            steps {
                sh '''
                echo "Stopping Odoo 17..."
                docker compose -f docker/docker-compose-odoo17.yml down -v

                docker system prune -af --volumes || true
                df -h
                '''
            }
        }

        stage('Start Odoo 18') {
            steps {
                sh 'docker compose -f docker/docker-compose-odoo18.yml up -d --remove-orphans'
            }
        }

        stage('Wait for Odoo 18 DB') {
            steps {
                sh '''
                until docker exec ${ODOO18_DB_CONTAINER} pg_isready -U ${DB_USER}; do
                    sleep 5
                done
                '''
            }
        }

        stage('Drop & Recreate Odoo 18 DB (SAFE)') {
            steps {
                sh '''
                echo "Dropping existing Odoo 18 DB safely..."

                docker exec ${ODOO18_DB_CONTAINER} psql -U ${DB_USER} -d postgres -c "
                SELECT pg_terminate_backend(pid)
                FROM pg_stat_activity
                WHERE datname = '${ODOO18_DB}';
                "

                docker exec ${ODOO18_DB_CONTAINER} psql -U ${DB_USER} -d postgres \
                    -c "DROP DATABASE IF EXISTS ${ODOO18_DB};"

                docker exec ${ODOO18_DB_CONTAINER} psql -U ${DB_USER} -d postgres \
                    -c "CREATE DATABASE ${ODOO18_DB} OWNER ${DB_USER};"
                '''
            }
        }

        stage('Restore DB into Odoo 18') {
            steps {
                sh '''
                echo "Restoring dump into Odoo 18..."
                docker cp ${ODOO17_DUMP} ${ODOO18_DB_CONTAINER}:/tmp/${ODOO17_DUMP}

                docker exec ${ODOO18_DB_CONTAINER} pg_restore \
                    -U ${DB_USER} \
                    -d ${ODOO18_DB} \
                    --no-owner \
                    --no-acl \
                    /tmp/${ODOO17_DUMP}
                '''
            }
        }

        /* =======================
           FINAL CLEANUP
        ======================= */
        stage('Final Cleanup 🧹') {
            steps {
                sh '''
                rm -f ${ODOO17_DUMP}
                docker system prune -f || true
                df -h
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Odoo 17 → Odoo 18 DB migration completed successfully"
        }
        failure {
            echo "❌ Migration failed — check Jenkins logs"
        }
    }
}

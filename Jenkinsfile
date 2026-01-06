pipeline {
    agent any

    environment {
        // Git
        GIT_REPO = 'https://github.com/SoujanyaKN/odoo-db-migration.git'
        BRANCH   = 'main'

        // Odoo 17
        ODOO17_DB_CONTAINER = 'odoo17-db'
        ODOO17_DB           = 'odoo17_db'
        ODOO17_DUMP         = 'odoo17.dump'

        // Odoo 18
        ODOO18_DB_CONTAINER = 'odoo18-db'
        ODOO18_DB           = 'odoo18_db'

        // DB creds
        DB_USER = 'odoo'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

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
                sh '''
                docker compose -f docker/docker-compose-odoo17.yml up -d
                '''
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
                docker exec ${ODOO17_DB_CONTAINER} pg_dump \
                  -U ${DB_USER} \
                  -Fc \
                  ${ODOO17_DB} > ${ODOO17_DUMP}
                '''
            }
        }

        stage('Start Odoo 18') {
            steps {
                sh '''
                docker compose -f docker/docker-compose-odoo18.yml up -d
                '''
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

        stage('Drop & Recreate Odoo 18 DB') {
            steps {
                sh '''
                docker exec ${ODOO18_DB_CONTAINER} psql -U ${DB_USER} -d postgres -c "
                SELECT pg_terminate_backend(pid)
                FROM pg_stat_activity
                WHERE datname = '${ODOO18_DB}';
                "

                docker exec ${ODOO18_DB_CONTAINER} psql -U ${DB_USER} -d postgres -c "DROP DATABASE IF EXISTS ${ODOO18_DB};"
                docker exec ${ODOO18_DB_CONTAINER} psql -U ${DB_USER} -d postgres -c "CREATE DATABASE ${ODOO18_DB} OWNER ${DB_USER};"
                '''
            }
        }

        stage('Restore DB into Odoo 18') {
            steps {
                sh '''
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

        stage('Cleanup') {
            steps {
                sh '''
                rm -f ${ODOO17_DUMP}
                docker system prune -f || true
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Odoo 17 → Odoo 18 DB restore completed successfully"
        }
        failure {
            echo "❌ Migration failed — check logs"
        }
    }
}

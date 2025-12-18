pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        // DB config
        DB_USER = 'odoo'
        DB_PASS = 'odoo'
        ODOO17_DB = 'odoo17_db'
        ODOO18_DB = 'odoo18_db'

        // Containers
        ODOO17_WEB = 'odoo17-web'
        ODOO17_DB_CONT = 'odoo17-db'
        ODOO18_WEB = 'odoo18-web'
        ODOO18_DB_CONT = 'odoo18-db'
    }

    stages {

        /* -------------------- STAGE 1 -------------------- */
        stage('Start Odoo 17 Pod') {
            steps {
                sh '''
                echo "Starting Odoo 17 pod..."
                docker-compose -f docker/docker-compose-odoo17.yml up -d
                '''
            }
        }

        /* -------------------- STAGE 2 -------------------- */
        stage('Wait for PostgreSQL & Initialize Odoo 17 DB') {
            steps {
                sh '''
                echo "Waiting for PostgreSQL to be ready..."
                until docker exec ${ODOO17_DB_CONT} pg_isready -U ${DB_USER} &>/dev/null; do
                    echo "PostgreSQL not ready yet, sleeping 5s..."
                    sleep 5
                done
                echo "PostgreSQL is ready."

                echo "Initializing Odoo 17 DB (base module)..."
                docker exec ${ODOO17_WEB} odoo -i base --db_host=db --db_user=${DB_USER} --db_password=${DB_PASS} --stop-after-init

                echo "Odoo 17 initialized successfully."
                docker exec ${ODOO17_WEB} odoo --version
                '''
            }
        }

        /* -------------------- STAGE 3 -------------------- */
        stage('Dump Odoo 17 Database') {
            steps {
                sh '''
                echo "Dumping Odoo 17 DB..."
                docker exec ${ODOO17_DB_CONT} pg_dump -U ${DB_USER} -F c ${ODOO17_DB} > odoo17.dump
                ls -lh odoo17.dump
                '''
            }
        }

        /* -------------------- STAGE 4 -------------------- */
        stage('Stop Odoo 17 Pod') {
            steps {
                sh '''
                echo "Stopping Odoo 17 pod..."
                docker-compose -f docker/docker-compose-odoo17.yml down
                '''
            }
        }

        /* -------------------- STAGE 5 -------------------- */
        stage('Start Odoo 18 Pod') {
            steps {
                sh '''
                echo "Starting Odoo 18 pod..."
                docker-compose -f docker/docker-compose-odoo18.yml up -d
                '''
            }
        }

        /* -------------------- STAGE 6 -------------------- */
        stage('Wait for PostgreSQL & Restore DB into Odoo 18') {
            steps {
                sh '''
                echo "Waiting for PostgreSQL to be ready..."
                until docker exec ${ODOO18_DB_CONT} pg_isready -U ${DB_USER} &>/dev/null; do
                    echo "PostgreSQL not ready yet, sleeping 5s..."
                    sleep 5
                done
                echo "PostgreSQL is ready."

                echo "Restoring DB into Odoo 18..."
                docker exec ${ODOO18_DB_CONT} dropdb -U ${DB_USER} ${ODOO18_DB} || true
                docker exec ${ODOO18_DB_CONT} createdb -U ${DB_USER} ${ODOO18_DB}

                docker exec -i ${ODOO18_DB_CONT} pg_restore -U ${DB_USER} -d ${ODOO18_DB} < odoo17.dump
                '''
            }
        }

        /* -------------------- STAGE 7 -------------------- */
        stage('Run OpenUpgrade (17 → 18)') {
            steps {
                sh '''
                echo "Running OpenUpgrade..."
                docker exec ${ODOO18_WEB} python odoo-bin \
                  -d ${ODOO18_DB} \
                  --upgrade-path=/mnt/openupgrade/openupgrade_scripts/scripts \
                  --update all \
                  --stop-after-init \
                  --load=base,web,openupgrade_framework
                '''
            }
        }

        /* -------------------- STAGE 8 -------------------- */
        stage('Verify Odoo 18 Version') {
            steps {
                sh '''
                echo "Checking Odoo 18 version..."
                docker exec ${ODOO18_WEB} odoo --version

                echo "Listing all modules in Odoo 18 DB:"
                docker exec ${ODOO18_DB_CONT} psql -U ${DB_USER} -d ${ODOO18_DB} \
                  -c "SELECT name, latest_version FROM ir_module_module ORDER BY name;" || true
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Odoo DB Migration 17 → 18 completed successfully"
        }
        failure {
            echo "❌ Migration failed. Check logs."
        }
        always {
            sh 'docker ps'
        }
    }
}

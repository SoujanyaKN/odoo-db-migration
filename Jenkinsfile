pipeline {
    agent any

    options {
        timestamps()
        //ansiColor('xterm')
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
                sleep 30
                '''
            }
        }

        /* -------------------- STAGE 2 -------------------- */
        stage('Verify Odoo 17 Version') {
            steps {
                sh '''
                echo "Waiting for Odoo 17 DB to be ready..."
                until docker exec ${ODOO17_WEB} odoo --stop-after-init &>/dev/null; do
                    echo "Odoo 17 not ready yet, sleeping 10s..."
                    sleep 10
                done

                echo "Odoo 17 version:"
                docker exec ${ODOO17_WEB} odoo --version

                echo "Checking DB base module version..."
                docker exec ${ODOO17_DB_CONT} psql -U ${DB_USER} -d ${ODOO17_DB} \
                  -c "SELECT latest_version FROM ir_module_module WHERE name='base';"
                '''
            }
        }

        /* -------------------- STAGE 3 -------------------- */
        stage('Dump Odoo 17 Database') {
            steps {
                sh '''
                echo "Dumping Odoo 17 DB..."
                docker exec ${ODOO17_DB_CONT} pg_dump \
                  -U ${DB_USER} -F c ${ODOO17_DB} > odoo17.dump

                ls -lh odoo17.dump
                '''
            }
        }

        /* -------------------- STAGE 4 -------------------- */
        stage('Stop Odoo 17 Pod') {
            steps {
                sh '''
                echo "Stopping Odoo 17 pod..."
                docker-compose -f docker-compose-odoo17.yml down
                '''
            }
        }

        /* -------------------- STAGE 5 -------------------- */
        stage('Start Odoo 18 Pod') {
            steps {
                sh '''
                echo "Starting Odoo 18 pod..."
                docker-compose -f docker/docker-compose-odoo18.yml up -d
                sleep 30
                '''
            }
        }

        /* -------------------- STAGE 6 -------------------- */
        stage('Restore DB into Odoo 18') {
            steps {
                sh '''
                echo "Restoring DB into Odoo 18..."

                docker exec ${ODOO18_DB_CONT} dropdb -U ${DB_USER} ${ODOO18_DB} || true
                docker exec ${ODOO18_DB_CONT} createdb -U ${DB_USER} ${ODOO18_DB}

                docker exec -i ${ODOO18_DB_CONT} pg_restore \
                  -U ${DB_USER} -d ${ODOO18_DB} < odoo17.dump
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

                echo "Checking DB base module version..."
                docker exec ${ODOO18_DB_CONT} psql -U ${DB_USER} -d ${ODOO18_DB} \
                  -c "SELECT latest_version FROM ir_module_module WHERE name='base';"
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

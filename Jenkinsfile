pipeline {
    agent any
    options { 
        timestamps()
    }

    environment {
        // ---------- Database credentials ----------
        DB_USER     = 'odoo'
        DB_PASSWORD = 'odoo'
        DB_PORT     = '5432'

        // ---------- Database hosts ----------
        ODOO17_DB_HOST = 'odoo17-db'
        ODOO18_DB_HOST = 'odoo18-db'

        // ---------- Databases ----------
        ODOO17_DB = 'odoo17_db'
        ODOO18_DB = 'odoo18_db'

        // ---------- Dump ----------
        ODOO17_DUMP = 'odoo17.dump'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Start Odoo 17 Pod') {
            steps {
                sh '''
                echo "Starting Odoo 17 containers..."
                docker-compose -f docker/docker-compose-odoo17.yml up -d
                '''
            }
        }

        stage('Wait & Initialize Odoo 17 DB') {
            steps {
                sh '''
                echo "Waiting for Odoo 17 PostgreSQL..."
                until docker exec ${ODOO17_DB_HOST} pg_isready -U ${DB_USER} -h ${ODOO17_DB_HOST} -p ${DB_PORT}; do
                    sleep 5
                done

                echo "Initializing Odoo 17 database with core modules..."
                docker exec odoo17-web odoo \
                    -d ${ODOO17_DB} \
                    -i base,web,mail,board,account,stock,sale,purchase \
                    --db_host=${ODOO17_DB_HOST} \
                    --db_port=${DB_PORT} \
                    --db_user=${DB_USER} \
                    --db_password=${DB_PASSWORD} \
                    --stop-after-init
                '''
            }
        }

        stage('Dump Odoo 17 Database') {
            steps {
                sh '''
                echo "Dumping Odoo 17 database..."
                docker exec ${ODOO17_DB_HOST} pg_dump -U ${DB_USER} -F c ${ODOO17_DB} > ${ODOO17_DUMP}
                ls -lh ${ODOO17_DUMP}
                '''
            }
        }

        stage('Stop & Clean Odoo 17 (Free Disk Space)') {
            steps {
                sh '''
                echo "Stopping Odoo 17 containers and freeing disk space..."
                docker-compose -f docker/docker-compose-odoo17.yml down -v
                docker system prune -af --volumes
                df -h
                '''
            }
        }

        stage('Start Odoo 18 Pod') {
            steps {
                sh '''
                echo "Starting Odoo 18 containers..."
                docker-compose -f docker/docker-compose-odoo18.yml up -d --remove-orphans
                '''
            }
        }

        stage('Wait for Odoo 18 DB') {
            steps {
                sh '''
                echo "Waiting for Odoo 18 PostgreSQL..."
                until docker exec ${ODOO18_DB_HOST} pg_isready -U ${DB_USER} -h ${ODOO18_DB_HOST} -p ${DB_PORT}; do
                    sleep 5
                done
                '''
            }
        }

        stage('Restore Dump into Odoo 18') {
            steps {
                sh '''
                echo "Checking if Odoo 18 DB is empty..."
                COUNT=$(docker exec ${ODOO18_DB_HOST} psql -U ${DB_USER} -d ${ODOO18_DB} -tAc "SELECT count(*) FROM pg_tables WHERE schemaname='public';")
                if [ "$COUNT" -eq "0" ]; then
                    echo "Odoo 18 DB is empty, skipping restore."
                else
                    echo "Restoring Odoo 17 dump into Odoo 18 DB..."
                    docker exec -i ${ODOO18_DB_HOST} pg_restore \
                        -U ${DB_USER} \
                        -d ${ODOO18_DB} \
                        --clean \
                        --if-exists \
                        < ${ODOO17_DUMP} || true
                fi
                '''
            }
        }

        stage('Run OpenUpgrade Migration') {
            steps {
                sh '''
                echo "Running OpenUpgrade migration..."
                docker exec odoo18-web odoo \
                    -d ${ODOO18_DB} \
                    -u all \
                    --db_host=${ODOO18_DB_HOST} \
                    --db_port=${DB_PORT} \
                    --db_user=${DB_USER} \
                    --db_password=${DB_PASSWORD} \
                    --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade \
                    --stop-after-init
                '''
            }
        }
    }

    post {
        always {
            sh '''
            echo "Final cleanup: stopping containers"
            docker-compose -f docker/docker-compose-odoo17.yml down || true
            docker-compose -f docker/docker-compose-odoo18.yml down || true
            '''
        }

        success {
            echo "✅ Odoo 17 → Odoo 18 migration completed successfully"
        }

        failure {
            echo "❌ Migration failed — check logs"
        }
    }
}

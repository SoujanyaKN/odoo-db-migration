pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        DB_USER     = 'odoo'
        DB_PASSWORD = 'odoo'
        DB_PORT     = '5432'

        ODOO17_DB_HOST = 'odoo17-db'
        ODOO18_DB_HOST = 'odoo18-db'

        ODOO17_DB = 'odoo17_db'
        ODOO18_DB = 'odoo18_db'

        ODOO17_DUMP = 'odoo17.dump'
    }

    stages {

        stage('Cleanup Old Containers & Volumes') {
            steps {
                sh '''
                    echo "Removing old Odoo containers and volumes if any..."
                    docker rm -f odoo17-web odoo17-db odoo18-web odoo18-db || true
                    docker volume rm odoo17-db odoo18-db || true
                    docker system prune -af --volumes
                    echo "Disk and memory:"
                    df -h
                    free -h
                '''
            }
        }

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
                    docker ps --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"
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

                    echo "Creating base Odoo 17 database and core modules..."
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

        stage('Dump Odoo 17 Database (stream preferred)') {
            steps {
                sh '''
                    echo "Before dump:"
                    free -h

                    docker exec -i ${ODOO17_DB_HOST} pg_dump \
                        -U ${DB_USER} \
                        -F c ${ODOO17_DB} > ${ODOO17_DUMP}

                    ls -lh ${ODOO17_DUMP}
                '''
            }
        }

        stage('Stop & Clean Odoo 17') {
            steps {
                sh '''
                    echo "Stopping Odoo 17 containers..."
                    docker-compose -f docker/docker-compose-odoo17.yml down -v
                    docker system prune -af --volumes
                    echo "Disk and memory after cleanup:"
                    df -h
                    free -h
                '''
            }
        }

        stage('Prepare OpenUpgrade 18 sources') {
            steps {
                sh '''
                    echo "Preparing OpenUpgrade 18.0..."
                    if [ ! -d docker/OpenUpgrade-18.0 ]; then
                        git clone --depth 1 --branch 18.0 \
                            https://github.com/OCA/OpenUpgrade.git docker/OpenUpgrade-18.0
                    else
                        echo "OpenUpgrade already present."
                    fi

                    test -f docker/OpenUpgrade-18.0/openupgrade_framework/__manifest__.py
                '''
            }
        }

        stage('Start Odoo 18 Pod') {
            steps {
                sh '''
                    echo "Starting Odoo 18 containers..."
                    docker-compose -f docker/docker-compose-odoo18.yml up -d --remove-orphans
                    docker ps --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"
                '''
            }
        }

        stage('Wait for Odoo 18 DB') {
            steps {
                sh '''
                    echo "Waiting for Odoo 18 PostgreSQL..."
                    until docker exec ${ODOO18_DB_HOST} pg_isready \
                        -U ${DB_USER} \
                        -h ${ODOO18_DB_HOST} \
                        -p ${DB_PORT}; do
                        sleep 5
                    done
                '''
            }
        }

        stage('Restore Dump into Odoo 18') {
            steps {
                sh '''
                    echo "Restoring Odoo 17 dump into Odoo 18 DB..."
                    cat ${ODOO17_DUMP} | docker exec -i ${ODOO18_DB_HOST} pg_restore \
                        -U ${DB_USER} \
                        -d ${ODOO18_DB} \
                        --clean \
                        --if-exists
                '''
            }
        }

        stage('Run OpenUpgrade Migration') {
            steps {
                sh '''
                    echo "Running OpenUpgrade migration with framework loaded..."
                    docker exec odoo18-web odoo \
                        -d ${ODOO18_DB} \
                        --db_host=${ODOO18_DB_HOST} \
                        --db_port=${DB_PORT} \
                        --db_user=${DB_USER} \
                        --db_password=${DB_PASSWORD} \
                        --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_framework \
                        --load=base,web,openupgrade_framework \
                        -u all \
                        --stop-after-init
                '''
            }
        }

        stage('Post checks') {
            steps {
                sh '''
                    echo "Containers running and ports:"
                    docker ps --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"
                    echo "Memory state:"
                    free -h
                    echo "Open UI at http://<EC2-IP>:8070"
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Odoo 17 → Odoo 18 migration completed. UI is on port 8070; Jenkins stays on 8080."
        }
        failure {
            echo "❌ Migration failed — check the migration stage logs."
        }
    }
}

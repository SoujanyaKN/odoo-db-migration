pipeline {
    agent any
    options { 
        timestamps()
    }

    environment {
        // Odoo 17 DB
        ODOO17_DB = 'odoo17_db'
        DB_USER = 'odoo'
        DB_PASSWORD = 'odoo'
        DB_HOST = 'db'
        DB_PORT = '5432'
        // Dump file
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

        stage('Wait for PostgreSQL & Initialize Odoo 17 DB') {
            steps {
                script {
                    echo "Waiting for PostgreSQL to be ready..."
                    sh '''
                    until docker exec odoo17-db pg_isready -U ${DB_USER} -h ${DB_HOST} -p ${DB_PORT}; do
                        echo "Waiting for PostgreSQL..."
                        sleep 5
                    done
                    echo "PostgreSQL is ready."
                    '''

                    echo "Initializing Odoo 17 DB with all modules..."
                    sh '''
                    docker exec odoo17-web odoo -d ${ODOO17_DB} \
                        -i base,web,mail,board,account,stock,sale,purchase \
                        --db_host=${DB_HOST} \
                        --db_port=${DB_PORT} \
                        --db_user=${DB_USER} \
                        --db_password=${DB_PASSWORD} \
                        --stop-after-init
                    '''
                    echo "Odoo 17 DB initialized successfully."
                }
            }
        }

        stage('Dump Odoo 17 Database') {
            steps {
                sh '''
                echo "Dumping Odoo 17 database..."
                docker exec odoo17-db pg_dump -U ${DB_USER} -F c ${ODOO17_DB} > ${ODOO17_DUMP}
                ls -lh ${ODOO17_DUMP}
                '''
            }
        }

        stage('Stop Odoo 17 Pod') {
            steps {
                sh '''
                echo "Stopping Odoo 17 containers..."
                docker-compose -f docker/docker-compose-odoo17.yml down
                '''
            }
        }

        stage('Start Odoo 18 Pod') {
            steps {
                sh '''
                echo "Starting Odoo 18 containers..."
                docker-compose -f docker/docker-compose-odoo18.yml up -d
                '''
            }
        }

        stage('Restore & Initialize Odoo 18 DB') {
            steps {
                script {
                    echo "Waiting for Odoo 18 PostgreSQL to be ready..."
                    sh '''
                    until docker exec odoo18-db pg_isready -U ${DB_USER} -h ${DB_HOST} -p ${DB_PORT}; do
                        echo "Waiting for PostgreSQL..."
                        sleep 5
                    done
                    echo "PostgreSQL is ready."
                    '''

                    echo "Restoring Odoo 17 dump into Odoo 18..."
                    sh '''
                    docker exec -i odoo18-db pg_restore -U ${DB_USER} -d ${ODOO17_DB} --clean < ${ODOO17_DUMP}
                    '''

                    echo "Initializing Odoo 18 DB (migrate modules)..."
                    sh '''
                    docker exec odoo18-web odoo -d ${ODOO17_DB} -u all \
                        --db_host=${DB_HOST} \
                        --db_port=${DB_PORT} \
                        --db_user=${DB_USER} \
                        --db_password=${DB_PASSWORD} \
                        --stop-after-init
                    '''
                    echo "Odoo 18 DB initialized successfully."
                }
            }
        }
    }

    post {
        always {
            sh '''
            echo "Cleanup: ensure containers are down"
            docker-compose -f docker/docker-compose-odoo17.yml down || true
            docker-compose -f docker/docker-compose-odoo18.yml down || true
            '''
        }
    }
}

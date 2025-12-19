pipeline {
    agent any
    options { 
        timestamps()
        //ansiColor('xterm')
    }

    environment {
<<<<<<< HEAD
        // Odoo 17 DB
        ODOO17_DB = 'odoo17_db'
        DB_USER = 'odoo'
        DB_PASSWORD = 'odoo'
        DB_HOST = 'db'
        DB_PORT = '5432'
        // Dump file
=======
        // ---------- Database credentials ----------
        DB_USER     = 'odoo'
        DB_PASSWORD = 'odoo'
        DB_PORT     = '5432'

        // ---------- Databases ----------
        ODOO17_DB = 'odoo17_db'
        ODOO18_DB = 'odoo18_db'

        // ---------- Dump ----------
>>>>>>> 5d1e7ce (Update Jenkins pipeline for Odoo 17 → 18 migration)
        ODOO17_DUMP = 'odoo17.dump'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

<<<<<<< HEAD
        stage('Start Odoo 17 Pod') {
=======
        /* ---------------------------------------------------------- */
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        /* ---------------------------------------------------------- */
        stage('Start Odoo 17') {
>>>>>>> 5d1e7ce (Update Jenkins pipeline for Odoo 17 → 18 migration)
            steps {
                sh '''
                echo "Starting Odoo 17 containers..."
                docker-compose -f docker/docker-compose-odoo17.yml up -d
                '''
            }
        }

<<<<<<< HEAD
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

=======
        /* ---------------------------------------------------------- */
        stage('Wait & Initialize Odoo 17 DB') {
            steps {
                sh '''
                echo "Waiting for Odoo 17 PostgreSQL..."
                until docker exec odoo17-db pg_isready -U ${DB_USER} -h localhost -p ${DB_PORT}; do
                    sleep 5
                done

                echo "Initializing Odoo 17 database with core modules..."
                docker exec odoo17-web odoo \
                    -d ${ODOO17_DB} \
                    -i base,web,mail,account,stock,sale,purchase \
                    --db_host=localhost \
                    --db_port=${DB_PORT} \
                    --db_user=${DB_USER} \
                    --db_password=${DB_PASSWORD} \
                    --stop-after-init
                '''
            }
        }

        /* ---------------------------------------------------------- */
>>>>>>> 5d1e7ce (Update Jenkins pipeline for Odoo 17 → 18 migration)
        stage('Dump Odoo 17 Database') {
            steps {
                sh '''
                echo "Dumping Odoo 17 database..."
<<<<<<< HEAD
                docker exec odoo17-db pg_dump -U ${DB_USER} -F c ${ODOO17_DB} > ${ODOO17_DUMP}
=======
                docker exec odoo17-db pg_dump \
                    -U ${DB_USER} \
                    -F c \
                    ${ODOO17_DB} > ${ODOO17_DUMP}

>>>>>>> 5d1e7ce (Update Jenkins pipeline for Odoo 17 → 18 migration)
                ls -lh ${ODOO17_DUMP}
                '''
            }
        }

<<<<<<< HEAD
        stage('Stop Odoo 17 Pod') {
            steps {
                sh '''
                echo "Stopping Odoo 17 containers..."
                docker-compose -f docker/docker-compose-odoo17.yml down
=======
        /* ---------------------------------------------------------- */
        stage('Stop & Clean Odoo 17 (FREE DISK SPACE)') {
            steps {
                sh '''
                echo "Stopping Odoo 17 and removing volumes..."
                docker-compose -f docker/docker-compose-odoo17.yml down -v
                docker system prune -f
                df -h
>>>>>>> 5d1e7ce (Update Jenkins pipeline for Odoo 17 → 18 migration)
                '''
            }
        }

<<<<<<< HEAD
        stage('Start Odoo 18 Pod') {
=======
        /* ---------------------------------------------------------- */
        stage('Start Odoo 18') {
>>>>>>> 5d1e7ce (Update Jenkins pipeline for Odoo 17 → 18 migration)
            steps {
                sh '''
                echo "Starting Odoo 18 containers..."
                docker-compose -f docker/docker-compose-odoo18.yml up -d
                '''
            }
        }

<<<<<<< HEAD
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
=======
        /* ---------------------------------------------------------- */
        stage('Wait for Odoo 18 DB') {
            steps {
                sh '''
                echo "Waiting for Odoo 18 PostgreSQL..."
                until docker exec odoo18-db pg_isready -U ${DB_USER} -h localhost -p ${DB_PORT}; do
                    sleep 5
                done
                '''
            }
        }

        /* ---------------------------------------------------------- */
        stage('Restore Dump into Odoo 18') {
            steps {
                sh '''
                echo "Restoring Odoo 17 dump into Odoo 18 DB..."
                docker exec -i odoo18-db pg_restore \
                    -U ${DB_USER} \
                    -d ${ODOO18_DB} \
                    --clean \
                    < ${ODOO17_DUMP}
                '''
            }
        }

        /* ---------------------------------------------------------- */
        stage('Run OpenUpgrade Migration') {
            steps {
                sh '''
                echo "Running OpenUpgrade migration..."
                docker exec odoo18-web odoo \
                    -d ${ODOO18_DB} \
                    -u all \
                    --db_host=localhost \
                    --db_port=${DB_PORT} \
                    --db_user=${DB_USER} \
                    --db_password=${DB_PASSWORD} \
                    --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade \
                    --stop-after-init
                '''
>>>>>>> 5d1e7ce (Update Jenkins pipeline for Odoo 17 → 18 migration)
            }
        }
    }

    /* ---------------------------------------------------------- */
    post {
        always {
            sh '''
<<<<<<< HEAD
            echo "Cleanup: ensure containers are down"
            docker-compose -f docker/docker-compose-odoo17.yml down || true
            docker-compose -f docker/docker-compose-odoo18.yml down || true
            '''
=======
            echo "Final cleanup..."
            docker-compose -f docker/docker-compose-odoo18.yml down || true
            '''
        }

        success {
            echo "✅ Odoo 17 → Odoo 18 migration completed successfully"
        }

        failure {
            echo "❌ Migration failed — check logs"
>>>>>>> 5d1e7ce (Update Jenkins pipeline for Odoo 17 → 18 migration)
        }
    }
}

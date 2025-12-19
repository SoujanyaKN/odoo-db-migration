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
                    echo "Cleaning old containers & volumes"
                    docker rm -f odoo17-web odoo17-db odoo18-web odoo18-db || true
                    docker volume rm odoo17-db odoo18-db || true
                    docker system prune -af --volumes
                    df -h
                '''
            }
        }

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Start Odoo 17') {
            steps {
                sh '''
                    docker-compose -f docker/docker-compose-odoo17.yml up -d
                '''
            }
        }

        stage('Wait & Init Odoo 17') {
            steps {
                sh '''
                    until docker exec ${ODOO17_DB_HOST} pg_isready -U ${DB_USER}; do sleep 5; done

                    docker exec odoo17-web odoo \
                      -d ${ODOO17_DB} \
                      -i base,web,mail,account,stock,sale,purchase \
                      --db_host=${ODOO17_DB_HOST} \
                      --db_user=${DB_USER} \
                      --db_password=${DB_PASSWORD} \
                      --stop-after-init
                '''
            }
        }

        stage('Dump Odoo 17 DB (stream)') {
            steps {
                sh '''
                    docker exec -i ${ODOO17_DB_HOST} pg_dump \
                      -U ${DB_USER} -F c ${ODOO17_DB} > ${ODOO17_DUMP}
                    ls -lh ${ODOO17_DUMP}
                '''
            }
        }

        stage('Stop Odoo 17') {
            steps {
                sh '''
                    docker-compose -f docker/docker-compose-odoo17.yml down -v
                    docker system prune -af --volumes
                '''
            }
        }

        stage('Prepare OpenUpgrade 18 (shallow clone)') {
            steps {
                sh '''
                    if [ ! -d docker/OpenUpgrade-18.0 ]; then
                        git clone --depth 1 --branch 18.0 \
                          https://github.com/OCA/OpenUpgrade.git docker/OpenUpgrade-18.0
                    fi

                    mkdir -p docker/OpenUpgrade-18.0/addons
                    cp -r docker/OpenUpgrade-18.0/openupgrade_framework \
                          docker/OpenUpgrade-18.0/addons/

                    test -f docker/OpenUpgrade-18.0/addons/openupgrade_framework/__manifest__.py
                '''
            }
        }

        stage('Start Odoo 18') {
            steps {
                sh '''
                    docker-compose -f docker/docker-compose-odoo18.yml up -d --remove-orphans
                '''
            }
        }

        stage('Wait for Odoo 18 DB') {
            steps {
                sh '''
                    until docker exec ${ODOO18_DB_HOST} pg_isready -U ${DB_USER}; do sleep 5; done
                '''
            }
        }

        stage('Restore DB into Odoo 18') {
            steps {
                sh '''
                    cat ${ODOO17_DUMP} | docker exec -i ${ODOO18_DB_HOST} pg_restore \
                      -U ${DB_USER} -d ${ODOO18_DB} --clean --if-exists
                '''
            }
        }

        stage('Run OpenUpgrade Migration - Base First ✅') {
            steps {
                sh '''
                    echo "Running OpenUpgrade migration: base module first..."

                    docker exec odoo18-web odoo \
                      -d ${ODOO18_DB} \
                      --db_host=${ODOO18_DB_HOST} \
                      --db_user=${DB_USER} \
                      --db_password=${DB_PASSWORD} \
                      --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                      -u base \
                      --without-demo=all \
                      --stop-after-init
                '''
            }
        }

        stage('Run OpenUpgrade Migration - Remaining Modules ✅') {
            steps {
                sh '''
                    echo "Running OpenUpgrade migration: remaining modules..."

                    docker exec odoo18-web odoo \
                      -d ${ODOO18_DB} \
                      --db_host=${ODOO18_DB_HOST} \
                      --db_user=${DB_USER} \
                      --db_password=${DB_PASSWORD} \
                      --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                      -u web,mail,account,stock,sale,purchase \
                      --without-demo=all \
                      --stop-after-init
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Odoo 17 → Odoo 18 migration completed successfully"
        }
        failure {
            echo "❌ Migration failed — check logs"
        }
    }
}

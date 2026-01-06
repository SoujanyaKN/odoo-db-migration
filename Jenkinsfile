pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        DB_USER         = 'odoo'
        DB_PASSWORD     = 'odoo'
        DB_PORT         = '5432'

        ODOO17_DB_HOST  = 'odoo17-db'
        ODOO18_DB_HOST  = 'odoo18-db'

        ODOO17_DB       = 'odoo17_db'
        ODOO18_DB       = 'odoo18_db'

        ODOO17_DUMP     = 'odoo17.dump'
    }

    stages {

        stage('Cleanup 🧨') {
            steps {
                sh '''
                    docker rm -f odoo17-web odoo17-db odoo18-web odoo18-db || true
                    docker system prune -af --volumes
                    rm -f ${ODOO17_DUMP}
                    df -h
                '''
            }
        }

        stage('Checkout SCM') {
            steps { checkout scm }
        }

        stage('Start Odoo 17') {
            steps {
                sh 'docker compose -f docker/docker-compose-odoo17.yml up -d'
            }
        }

        stage('Init Odoo 17') {
            steps {
                sh '''
                    until docker exec ${ODOO17_DB_HOST} pg_isready -U ${DB_USER}; do sleep 5; done

                    docker exec odoo17-web odoo \
                        -d ${ODOO17_DB} \
                        -i base,web,mail,sale,purchase,stock,account \
                        --without-demo=all \
                        --db_host=${ODOO17_DB_HOST} \
                        --db_user=${DB_USER} \
                        --db_password=${DB_PASSWORD} \
                        --stop-after-init
                '''
            }
        }

        stage('Dump Odoo 17 DB') {
            steps {
                sh '''
                    docker exec ${ODOO17_DB_HOST} pg_dump \
                      -U ${DB_USER} -F c -f /tmp/${ODOO17_DUMP} ${ODOO17_DB}
                    docker cp ${ODOO17_DB_HOST}:/tmp/${ODOO17_DUMP} .
                '''
            }
        }

        stage('Free Disk 🧹') {
            steps {
                sh '''
                    docker compose -f docker/docker-compose-odoo17.yml down -v
                    docker system prune -af --volumes
                    df -h
                '''
            }
        }

        stage('Prepare OpenUpgrade 18') {
            steps {
                sh '''
                    if [ ! -d docker/OpenUpgrade-18.0 ]; then
                        git clone -b 18.0 https://github.com/OCA/OpenUpgrade.git docker/OpenUpgrade-18.0
                    fi
                '''
            }
        }

        stage('Start Odoo 18') {
            steps {
                sh 'docker compose -f docker/docker-compose-odoo18.yml up -d'
            }
        }

        stage('Restore DB into Odoo 18') {
            steps {
                sh '''
                    until docker exec ${ODOO18_DB_HOST} pg_isready -U ${DB_USER}; do sleep 5; done

                    docker cp ${ODOO17_DUMP} ${ODOO18_DB_HOST}:/tmp/${ODOO17_DUMP}

                    docker exec ${ODOO18_DB_HOST} pg_restore \
                      -U ${DB_USER} -d ${ODOO18_DB} \
                      --clean --if-exists --no-owner --no-acl \
                      /tmp/${ODOO17_DUMP}
                '''
            }
        }

        stage('OpenUpgrade – BASE ONLY ✅') {
            steps {
                sh '''
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

        stage('OpenUpgrade – CORE MODULES ✅') {
            steps {
                sh '''
                    docker exec odoo18-web odoo \
                      -d ${ODOO18_DB} \
                      --db_host=${ODOO18_DB_HOST} \
                      --db_user=${DB_USER} \
                      --db_password=${DB_PASSWORD} \
                      --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                      -u web,mail,product,uom \
                      --without-demo=all \
                      --stop-after-init
                '''
            }
        }

        stage('OpenUpgrade – BUSINESS MODULES ✅') {
            steps {
                sh '''
                    docker exec odoo18-web odoo \
                      -d ${ODOO18_DB} \
                      --db_host=${ODOO18_DB_HOST} \
                      --db_user=${DB_USER} \
                      --db_password=${DB_PASSWORD} \
                      --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                      -u sale,purchase,stock,account \
                      --without-demo=all \
                      --stop-after-init
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Odoo 17 → 18 migration SUCCESS"
        }
        failure {
            echo "❌ Migration FAILED — check logs"
        }
    }
}

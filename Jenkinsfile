pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        DB_USER     = 'odoo'
        DB_PASSWORD = 'odoo'

        ODOO17_DB = 'odoo17_db'
        ODOO18_DB = 'odoo18_db'

        ODOO17_DUMP = 'odoo17.dump'
    }

    stages {

        /* ================= CLEAN START ================= */

        stage('Initial Docker Cleanup') {
            steps {
                sh '''
                  docker rm -f odoo17-web odoo17-db odoo18-web odoo18-db openupgrade || true
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

        /* ================= ODOO 17 ================= */

        stage('Start Odoo 17') {
            steps {
                sh 'docker compose -f docker/docker-compose-odoo17.yml up -d'
            }
        }

        stage('Init Odoo 17 DB') {
            steps {
                sh '''
                  until docker exec odoo17-db pg_isready -U odoo; do sleep 5; done

                  docker exec odoo17-web odoo \
                    -d ${ODOO17_DB} \
                    -i base,web,mail,account,sale,purchase,stock \
                    --stop-after-init
                '''
            }
        }

        stage('Dump Odoo 17 DB') {
            steps {
                sh '''
                  docker exec -i odoo17-db pg_dump -U odoo -F c ${ODOO17_DB} > ${ODOO17_DUMP}
                  ls -lh ${ODOO17_DUMP}
                '''
            }
        }

        stage('Stop Odoo 17 & Free Disk') {
            steps {
                sh '''
                  docker compose -f docker/docker-compose-odoo17.yml down -v
                  docker system prune -af --volumes
                  df -h
                '''
            }
        }

        /* ================= OPENUPGRADE ================= */

        stage('Clone OpenUpgrade 18') {
            steps {
                sh '''
                  if [ ! -d OpenUpgrade ]; then
                    git clone --depth 1 --branch 18.0 https://github.com/OCA/OpenUpgrade.git
                  fi
                '''
            }
        }

        stage('Start Postgres for Odoo 18') {
            steps {
                sh '''
                  docker run -d \
                    --name odoo18-db \
                    -e POSTGRES_DB=postgres \
                    -e POSTGRES_USER=odoo \
                    -e POSTGRES_PASSWORD=odoo \
                    postgres:15
                  
                  until docker exec odoo18-db pg_isready -U odoo; do sleep 5; done
                '''
            }
        }

        stage('Create Empty Odoo 18 DB') {
            steps {
                sh '''
                  docker exec odoo18-db psql -U odoo -d postgres -c "CREATE DATABASE ${ODOO18_DB};"
                '''
            }
        }

        stage('Restore Odoo 17 Dump into Odoo 18 DB') {
            steps {
                sh '''
                  cat ${ODOO17_DUMP} | docker exec -i odoo18-db pg_restore \
                    -U odoo \
                    -d ${ODOO18_DB} \
                    --no-owner \
                    --no-privileges
                '''
            }
        }

        stage('OpenUpgrade - Base') {
            steps {
                sh '''
                  docker run --rm \
                    --name openupgrade \
                    --link odoo18-db:db \
                    -v $(pwd)/OpenUpgrade:/mnt/openupgrade \
                    odoo:18 \
                    odoo \
                      -d ${ODOO18_DB} \
                      --db_host=db \
                      --db_user=odoo \
                      --db_password=odoo \
                      --addons-path=/mnt/openupgrade/addons,/usr/lib/python3/dist-packages/odoo/addons \
                      -u base \
                      --without-demo=all \
                      --stop-after-init
                '''
            }
        }

        stage('OpenUpgrade - Remaining Modules') {
            steps {
                sh '''
                  docker run --rm \
                    --name openupgrade \
                    --link odoo18-db:db \
                    -v $(pwd)/OpenUpgrade:/mnt/openupgrade \
                    odoo:18 \
                    odoo \
                      -d ${ODOO18_DB} \
                      --db_host=db \
                      --db_user=odoo \
                      --db_password=odoo \
                      --addons-path=/mnt/openupgrade/addons,/usr/lib/python3/dist-packages/odoo/addons \
                      -u web,mail,account,sale,purchase,stock \
                      --without-demo=all \
                      --stop-after-init
                '''
            }
        }

        /* ================= FINAL ODOO 18 ================= */

        stage('Start Final Odoo 18') {
            steps {
                sh 'docker compose -f docker/docker-compose-odoo18.yml up -d'
            }
        }

        stage('Post Checks') {
            steps {
                sh '''
                  docker ps
                  echo "✅ Odoo 18 should now start without XML errors"
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Odoo 17 → Odoo 18 migration completed using OpenUpgrade'
        }
        failure {
            echo '❌ Migration failed — check logs'
        }
    }
}

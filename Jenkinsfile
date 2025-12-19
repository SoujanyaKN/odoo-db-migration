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

        stage('Cleanup Docker') {
            steps {
                sh '''
                  docker rm -f odoo17-web odoo17-db odoo18-web odoo18-db || true
                  docker volume rm odoo17-db odoo18-db || true
                  docker system prune -af --volumes
                '''
            }
        }

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        /* ---------------- ODOO 17 ---------------- */

        stage('Start Odoo 17') {
            steps {
                sh 'docker-compose -f docker/docker-compose-odoo17.yml up -d'
            }
        }

        stage('Init Dummy Data in Odoo 17') {
            steps {
                sh '''
                  until docker exec odoo17-db pg_isready -U odoo; do sleep 5; done

                  docker exec odoo17-web odoo \
                    -d odoo17_db \
                    -i base,web,mail,account,sale,purchase,stock \
                    --db_host=odoo17-db \
                    --db_user=odoo \
                    --db_password=odoo \
                    --stop-after-init
                '''
            }
        }

        stage('Dump Odoo 17 DB') {
            steps {
                sh '''
                  docker exec -i odoo17-db pg_dump -U odoo -F c odoo17_db > odoo17.dump
                  ls -lh odoo17.dump
                '''
            }
        }

        stage('Stop Odoo 17') {
            steps {
                sh '''
                  docker-compose -f docker/docker-compose-odoo17.yml down -v
                '''
            }
        }

        /* ---------------- PREP OPENUPGRADE ---------------- */

        stage('Prepare OpenUpgrade 18') {
            steps {
                sh '''
                  if [ ! -d docker/OpenUpgrade-18.0 ]; then
                    git clone --depth 1 --branch 18.0 https://github.com/OCA/OpenUpgrade.git docker/OpenUpgrade-18.0
                  fi

                  mkdir -p docker/OpenUpgrade-18.0/addons
                  rsync -a docker/OpenUpgrade-18.0/openupgrade_framework/ docker/OpenUpgrade-18.0/addons/openupgrade_framework/
                '''
            }
        }

        /* ---------------- ODOO 18 ---------------- */

        stage('Start Odoo 18 DB Only') {
            steps {
                sh '''
                  docker-compose -f docker/docker-compose-odoo18.yml up -d odoo18-db
                  until docker exec odoo18-db pg_isready -U odoo; do sleep 5; done
                '''
            }
        }

        stage('Recreate Empty Odoo 18 DB') {
            steps {
                sh '''
                  docker exec -i odoo18-db psql -U odoo <<EOF
                  DROP DATABASE IF EXISTS odoo18_db;
                  CREATE DATABASE odoo18_db OWNER odoo;
                  EOF
                '''
            }
        }

        stage('Restore Odoo 17 Dump into EMPTY DB') {
            steps {
                sh '''
                  cat odoo17.dump | docker exec -i odoo18-db pg_restore \
                    -U odoo \
                    -d odoo18_db \
                    --no-owner \
                    --no-privileges
                '''
            }
        }

        stage('Start Odoo 18 Web') {
            steps {
                sh 'docker-compose -f docker/docker-compose-odoo18.yml up -d odoo18-web'
            }
        }

        stage('OpenUpgrade - Base Module') {
            steps {
                sh '''
                  docker exec odoo18-web odoo \
                    -d odoo18_db \
                    --db_host=odoo18-db \
                    --db_user=odoo \
                    --db_password=odoo \
                    --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                    -u base \
                    --without-demo=all \
                    --stop-after-init
                '''
            }
        }

        stage('OpenUpgrade - Remaining Modules') {
            steps {
                sh '''
                  docker exec odoo18-web odoo \
                    -d odoo18_db \
                    --db_host=odoo18-db \
                    --db_user=odoo \
                    --db_password=odoo \
                    --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                    -u web,mail,account,sale,purchase,stock \
                    --without-demo=all \
                    --stop-after-init
                '''
            }
        }

        stage('Post Checks') {
            steps {
                sh '''
                  docker ps
                  echo "Open browser → Settings → About → Version should be 18"
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Odoo 17 → Odoo 18 migration completed successfully'
        }
        failure {
            echo '❌ Migration failed – check logs'
        }
    }
}

pipeline {
    agent any

    options { timestamps() }

    environment {
        DB_USER     = 'odoo'
        DB_PASSWORD = 'odoo'

        ODOO17_DB_HOST = 'odoo17-db'
        ODOO18_DB_HOST = 'odoo18-db'

        ODOO17_DB = 'odoo17_db'
        ODOO18_DB = 'odoo18_db'
    }

    stages {

        stage('Initial Docker Cleanup') {
            steps {
                sh '''
                  docker rm -f odoo17-web odoo17-db odoo18-web odoo18-db || true
                  docker volume rm odoo17-db odoo18-db || true
                  docker system prune -af --volumes
                  df -h
                '''
            }
        }

        stage('Checkout SCM') { steps { checkout scm } }

        stage('Start Odoo 17') {
            steps {
                sh 'docker compose -f docker/docker-compose-odoo17.yml up -d'
            }
        }

        stage('Init Dummy Data in Odoo 17') {
            steps {
                sh '''
                  until docker exec ${ODOO17_DB_HOST} pg_isready -U ${DB_USER}; do sleep 5; done
                  docker exec odoo17-web odoo \
                    -d ${ODOO17_DB} \
                    -i base,web,mail,account,sale,purchase,stock \
                    --db_host=${ODOO17_DB_HOST} \
                    --db_user=${DB_USER} \
                    --db_password=${DB_PASSWORD} \
                    --stop-after-init
                '''
            }
        }

        stage('Stream Odoo 17 → Odoo 18') {
            steps {
                sh '''
                  echo "Streaming Odoo 17 DB directly into Odoo 18..."
                  
                  docker compose -f docker/docker-compose-odoo18.yml up -d
                  until docker exec ${ODOO18_DB_HOST} pg_isready -U ${DB_USER}; do sleep 5; done

                  docker exec -i ${ODOO18_DB_HOST} psql -U ${DB_USER} -d postgres <<EOF
                  DROP DATABASE IF EXISTS ${ODOO18_DB};
                  CREATE DATABASE ${ODOO18_DB} OWNER ${DB_USER};
                  EOF

                  docker exec ${ODOO17_DB_HOST} pg_dump -U ${DB_USER} -F c ${ODOO17_DB} | \
                  docker exec -i ${ODOO18_DB_HOST} pg_restore -U ${DB_USER} -d ${ODOO18_DB} --no-owner --no-privileges --clean
                  
                  echo "✅ DB streamed successfully, no dump stored on disk"
                '''
            }
        }

        stage('Stop & Purge Odoo 17 (Free Disk)') {
            steps {
                sh '''
                  docker compose -f docker/docker-compose-odoo17.yml down -v
                  docker rm -f odoo17-web odoo17-db || true
                  docker volume rm odoo17-db || true
                  docker system prune -af --volumes
                  df -h
                '''
            }
        }

        stage('Prepare OpenUpgrade 18') {
            steps {
                sh '''
                  if [ ! -d docker/OpenUpgrade-18.0 ]; then
                    git clone --depth 1 --branch 18.0 https://github.com/OCA/OpenUpgrade.git docker/OpenUpgrade-18.0
                  fi
                  mkdir -p docker/OpenUpgrade-18.0/addons
                  rsync -a --delete docker/OpenUpgrade-18.0/openupgrade_framework/ docker/OpenUpgrade-18.0/addons/openupgrade_framework/
                '''
            }
        }

        stage('Patch <list> to <tree>') {
            steps {
                sh '''
                  OPENUPGRADE_DIR="docker/OpenUpgrade-18.0/addons"
                  BACKUP_DIR="${OPENUPGRADE_DIR}_backup_$(date +%Y%m%d_%H%M%S)"
                  cp -r "$OPENUPGRADE_DIR" "$BACKUP_DIR"

                  find "$OPENUPGRADE_DIR" -type f -name "*.xml" -print0 | while IFS= read -r -d '' file; do
                      awk "/<tree/ {in_tree=1} /<\\/tree>/ {in_tree=0} in_tree && /<list / {gsub(\"<list\",\"<tree\")} in_tree && /<\\/list>/ {gsub(\"</list>\",\"</tree\")} {print}" "$file" > "${file}.tmp" && mv "${file}.tmp" "$file"
                  done
                  echo "✅ Safe patch done, backup at $BACKUP_DIR"
                '''
            }
        }

        stage('OpenUpgrade - Base Module') {
            steps {
                sh '''
                  docker exec odoo18-web odoo -d ${ODOO18_DB} \
                    --db_host=${ODOO18_DB_HOST} --db_user=${DB_USER} --db_password=${DB_PASSWORD} \
                    --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                    -u base --without-demo=all --stop-after-init
                '''
            }
        }

        stage('OpenUpgrade - Remaining Modules') {
            steps {
                sh '''
                  docker exec odoo18-web odoo -d ${ODOO18_DB} \
                    --db_host=${ODOO18_DB_HOST} --db_user=${DB_USER} --db_password=${DB_PASSWORD} \
                    --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                    -u web,mail,account,sale,purchase,stock --without-demo=all --stop-after-init
                '''
            }
        }

        stage('Post Checks') {
            steps {
                sh '''
                  docker ps
                  echo "✅ Check Odoo 18 version in browser → Settings → About"
                  df -h
                '''
            }
        }
    }

    post {
        success { echo '✅ Migration completed successfully' }
        failure { echo '❌ Migration failed — check logs' }
    }
}

pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        DB_USER     = 'odoo'
        DB_PASSWORD = 'odoo'

        ODOO17_DB_HOST = 'odoo17-db'
        ODOO18_DB_HOST = 'odoo18-db'

        ODOO17_DB = 'odoo17_db'
        ODOO18_DB = 'odoo18_db'

        ODOO17_DUMP = 'odoo17.dump'
    }

    stages {

        /* ================= CLEAN START ================= */

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

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        /* ================= ODOO 17 ================= */

        stage('Start Odoo 17') {
            steps {
                sh 'docker-compose -f docker/docker-compose-odoo17.yml up -d'
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

        stage('Dump Odoo 17 DB') {
            steps {
                sh '''
                  echo "Dumping Odoo 17 database..."
                  docker exec -i ${ODOO17_DB_HOST} pg_dump -U ${DB_USER} -F c ${ODOO17_DB} > ${ODOO17_DUMP}
                  ls -lh ${ODOO17_DUMP}
                '''
            }
        }

        /* ===== FREE SPACE AFTER DUMP ===== */

        stage('Stop & Purge Odoo 17 (FREE DISK)') {
            steps {
                sh '''
                  echo "Stopping Odoo 17 and cleaning disk..."
                  docker-compose -f docker/docker-compose-odoo17.yml down -v
                  docker rm -f odoo17-web odoo17-db || true
                  docker volume rm odoo17-db || true
                  docker system prune -af --volumes
                  df -h
                '''
            }
        }

        /* ================= OPENUPGRADE ================= */

        stage('Prepare OpenUpgrade 18') {
            steps {
                sh '''
                  if [ ! -d docker/OpenUpgrade-18.0 ]; then
                    git clone --depth 1 --branch 18.0 https://github.com/OCA/OpenUpgrade.git docker/OpenUpgrade-18.0
                  fi

                  mkdir -p docker/OpenUpgrade-18.0/addons
                  rsync -a --delete \
                    docker/OpenUpgrade-18.0/openupgrade_framework/ \
                    docker/OpenUpgrade-18.0/addons/openupgrade_framework/
                '''
            }
        }

        /* ================= ODOO 18 ================= */

        stage('Start Odoo 18') {
            steps {
                sh '''
                  docker-compose -f docker/docker-compose-odoo18.yml up -d
                  until docker exec ${ODOO18_DB_HOST} pg_isready -U ${DB_USER}; do sleep 5; done
                '''
            }
        }

        stage('Recreate EMPTY Odoo 18 DB') {
            steps {
                sh '''
                  docker exec -i ${ODOO18_DB_HOST} psql -U ${DB_USER} -d postgres <<EOF
                  DROP DATABASE IF EXISTS ${ODOO18_DB};
                  CREATE DATABASE ${ODOO18_DB} OWNER ${DB_USER};
                  EOF
                '''
            }
        }

        stage('Restore Odoo 17 Dump into Odoo 18 DB') {
            steps {
                sh '''
                  echo "Restoring dump into Odoo 18 DB..."
                  cat ${ODOO17_DUMP} | docker exec -i ${ODOO18_DB_HOST} pg_restore \
                    -U ${DB_USER} \
                    -d ${ODOO18_DB} \
                    --no-owner \
                    --no-privileges
                '''
            }
        }

        /* ================= OPENUPGRADE CLEANUP ================= */

        stage('OpenUpgrade - Clean Base Views & Duplicate Languages') {
            steps {
                sh '''
                  echo "Cleaning base views and duplicate languages safely..."
                  docker exec -i ${ODOO18_DB_HOST} psql -U ${DB_USER} -d ${ODOO18_DB} -v ON_ERROR_STOP=1 <<SQL
BEGIN;

-- Disable FK temporarily
SET session_replication_role = replica;

-- Delete child views first
DELETE FROM ir_ui_view
WHERE inherit_id IS NOT NULL;

-- Delete base views
DELETE FROM ir_ui_view
WHERE id IN (
    SELECT res_id FROM ir_model_data
    WHERE model='ir.ui.view' AND module='base'
);

-- Delete corresponding ir_model_data entries
DELETE FROM ir_model_data
WHERE model='ir.ui.view' AND module='base';

-- Delete problematic duplicate languages
DELETE FROM res_lang
WHERE name IN ('Serbian (Cyrillic) / српски',
               'Belarusian / Беларусьская мова');

COMMIT;

-- Re-enable FK constraints
SET session_replication_role = DEFAULT;
SQL
                '''
            }
        }

        /* ================= OPENUPGRADE RUN ================= */

        stage('OpenUpgrade - Base Module') {
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

        stage('OpenUpgrade - Remaining Modules') {
            steps {
                sh '''
                  docker exec odoo18-web odoo \
                    -d ${ODOO18_DB} \
                    --db_host=${ODOO18_DB_HOST} \
                    --db_user=${DB_USER} \
                    --db_password=${DB_PASSWORD} \
                    --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                    -u web,mail,account,sale,purchase,stock \
                    --without-demo=all \
                    --stop-after-init
                '''
            }
        }

        /* ================= POST ================= */

        stage('Post Checks') {
            steps {
                sh '''
                  docker ps
                  echo "✅ Open browser → Settings → About → Version should show Odoo 18"
                  df -h
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Odoo 17 → Odoo 18 migration completed successfully'
        }
        failure {
            echo '❌ Migration failed — check Jenkins logs'
        }
    }
}

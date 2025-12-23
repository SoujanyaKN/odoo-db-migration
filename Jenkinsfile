pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        DB_USER         = 'odoo'
        DB_PASSWORD     = 'odoo'

        ODOO17_DB_HOST  = 'odoo17-db'
        ODOO18_DB_HOST  = 'odoo18-db'

        ODOO17_DB       = 'odoo17_db'
        ODOO18_DB       = 'odoo18_db'

        ODOO17_DUMP     = 'odoo17.dump'
    }

    stages {

        /* ================= CLEAN START ================= */

        stage('Total Environment Cleanup 🧨') {
            steps {
                sh '''
                    echo "Cleaning containers, volumes, and old dumps..."
                    docker rm -f odoo17-web odoo17-db odoo18-web odoo18-db || true
                    docker system prune -af --volumes
                    rm -f ${ODOO17_DUMP}
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

        stage('Wait & Init Odoo 17 (ALL modules)') {
            options { timeout(time: 30, unit: 'MINUTES') }
            steps {
                sh '''
                    echo "Waiting for Odoo 17 DB..."
                    until docker exec ${ODOO17_DB_HOST} pg_isready -U ${DB_USER}; do sleep 5; done

                    echo "Installing core business modules in Odoo 17..."
                    docker exec odoo17-web odoo \
                        -d ${ODOO17_DB} \
                        -i base,web,mail,account,sale,purchase,stock \
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
                    echo "Dumping Odoo 17 database..."
                    docker exec -i ${ODOO17_DB_HOST} pg_dump \
                        -U ${DB_USER} -F c -b -v -f /tmp/${ODOO17_DUMP} ${ODOO17_DB}
                    docker cp ${ODOO17_DB_HOST}:/tmp/${ODOO17_DUMP} ./
                    ls -lh ${ODOO17_DUMP}
                '''
            }
        }

        stage('Free Disk After Dump 🧹') {
            steps {
                sh '''
                    echo "Stopping Odoo 17 and freeing disk..."
                    docker compose -f docker/docker-compose-odoo17.yml down -v
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
                        git clone --depth 1 --branch 18.0 \
                          https://github.com/OCA/OpenUpgrade.git docker/OpenUpgrade-18.0
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
                sh 'docker compose -f docker/docker-compose-odoo18.yml up -d --remove-orphans'
            }
        }

        stage('Restore DB into Odoo 18') {
            steps {
                sh '''
                    echo "Waiting for Odoo 18 DB..."
                    until docker exec ${ODOO18_DB_HOST} pg_isready -U ${DB_USER}; do sleep 5; done

                    docker cp ${ODOO17_DUMP} ${ODOO18_DB_HOST}:/tmp/${ODOO17_DUMP}

                    echo "Restoring database into Odoo 18..."
                    docker exec -i ${ODOO18_DB_HOST} pg_restore \
                        -U ${DB_USER} \
                        -d ${ODOO18_DB} \
                        --clean --if-exists --no-owner --no-acl \
                        /tmp/${ODOO17_DUMP}
                '''
            }
        }

        /* ================= PRE-MIGRATION FIXES ================= */

        stage('Pre-OpenUpgrade DB Cleanup') {
            steps {
                sh '''
                    echo "Cleaning base views and duplicate languages..."

                    docker exec -i ${ODOO18_DB_HOST} psql -U ${DB_USER} -d ${ODOO18_DB} <<'EOSQL'
BEGIN;
SET session_replication_role = replica;

DELETE FROM ir_ui_view
WHERE id IN (
    SELECT res_id FROM ir_model_data
    WHERE model='ir.ui.view' AND module='base'
);

DELETE FROM ir_model_data
WHERE model='ir.ui.view' AND module='base';

DELETE FROM res_lang
WHERE name IN (
    'Serbian (Cyrillic) / српски',
    'Belarusian / Беларусьская мова'
);

SET session_replication_role = DEFAULT;
COMMIT;
EOSQL
                '''
            }
        }

        /* ================= OPENUPGRADE RUN ================= */

        stage('OpenUpgrade - Base Module ✅') {
            options { timeout(time: 30, unit: 'MINUTES') }
            steps {
                sh '''
                    docker exec -u 0 odoo18-web pip install openupgradelib --break-system-packages

                    docker exec odoo18-web odoo \
                        -d ${ODOO18_DB} \
                        --db_host=${ODOO18_DB_HOST} \
                        --db_user=${DB_USER} \
                        --db_password=${DB_PASSWORD} \
                        --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                        --load=base,web,openupgrade_framework \
                        -u base \
                        --without-demo=all \
                        --stop-after-init
                '''
            }
        }

        stage('OpenUpgrade - Remaining Modules ✅') {
            options { timeout(time: 30, unit: 'MINUTES') }
            steps {
                sh '''
                    docker exec odoo18-web odoo \
                        -d ${ODOO18_DB} \
                        --db_host=${ODOO18_DB_HOST} \
                        --db_user=${DB_USER} \
                        --db_password=${DB_PASSWORD} \
                        --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                        --load=base,web,openupgrade_framework \
                        -u web,mail,account,sale,purchase,stock \
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
            echo "❌ Migration failed — check Jenkins logs"
        }
    }
}

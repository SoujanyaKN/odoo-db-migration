pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 2, unit: 'HOURS') // Global safety timeout
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
        stage('Total Environment Cleanup 🧨') {
            steps {
                sh '''
                    echo "Cleaning all containers, volumes and old dumps..."
                    docker rm -f odoo17-web odoo17-db odoo18-web odoo18-db || true
                    docker system prune -f
                    rm -f ${ODOO17_DUMP}
                    df -h
                '''
            }
        }

        stage('Checkout SCM') {
            steps { checkout scm }
        }

        stage('Start Odoo 17') {
            steps { sh 'docker compose -f docker/docker-compose-odoo17.yml up -d' }
        }

        stage('Wait & Init Odoo 17') {
            steps {
                sh '''
                    echo "Waiting for Odoo 17 DB..."
                    until docker exec ${ODOO17_DB_HOST} pg_isready -U ${DB_USER}; do sleep 5; done

                    echo "Step 1: Initializing Base..."
                    docker exec odoo17-web odoo -d ${ODOO17_DB} -i base --stop-after-init --log-level=warn

                    echo "Step 2: Initializing Heavy Modules (Account, Stock, etc.)..."
                    docker exec odoo17-web odoo \
                        -d ${ODOO17_DB} \
                        -i sale,purchase,stock,account,mail,web \
                        --without-demo=all \
                        --db_host=${ODOO17_DB_HOST} \
                        --db_user=${DB_USER} \
                        --db_password=${DB_PASSWORD} \
                        --log-level=info \
                        --stop-after-init
                '''
            }
        }

        stage('Dump Odoo 17 DB') {
            steps {
                sh '''
                    echo "Dumping Odoo 17 database..."
                    docker exec -i ${ODOO17_DB_HOST} pg_dump -U ${DB_USER} -F c ${ODOO17_DB} > ${ODOO17_DUMP}
                    if [ ! -s ${ODOO17_DUMP} ]; then echo "ERROR: Dump file is empty"; exit 1; fi
                '''
            }
        }

        stage('Mid-Pipeline Disk Cleanup 🧹') {
            steps {
                sh '''
                    docker compose -f docker/docker-compose-odoo17.yml down -v
                    docker system prune -f
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

        stage('Start Odoo 18') {
            steps { sh 'docker compose -f docker/docker-compose-odoo18.yml up -d --remove-orphans' }
        }

        stage('Restore DB into Odoo 18') {
            steps {
                sh '''
                    echo "Waiting for Odoo 18 DB..."
                    until docker exec ${ODOO18_DB_HOST} pg_isready -U ${DB_USER}; do sleep 5; done

                    echo "Dropping auto-created Odoo 18 DB to avoid constraint conflicts..."
                    docker exec -i ${ODOO18_DB_HOST} psql -U ${DB_USER} -d postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = '${ODOO18_DB}';"
                    docker exec -i ${ODOO18_DB_HOST} dropdb -U ${DB_USER} --if-exists ${ODOO18_DB}
                    docker exec -i ${ODOO18_DB_HOST} createdb -U ${DB_USER} ${ODOO18_DB}

                    echo "Restoring Odoo 17 dump..."
                    cat ${ODOO17_DUMP} | docker exec -i ${ODOO18_DB_HOST} pg_restore \
                        -U ${DB_USER} -d ${ODOO18_DB} --no-owner --role=${DB_USER}
                '''
            }
        }

        stage('Run OpenUpgrade Migration - Base First ✅') {
            steps {
                sh '''
                    docker exec -u 0 odoo18-web pip install openupgradelib --break-system-packages

                    echo "SQL Cleanup..."
                    echo "
                    DELETE FROM ir_ui_view WHERE id IN (SELECT res_id FROM ir_model_data WHERE model='ir.ui.view' AND module='base');
                    DELETE FROM ir_model_data WHERE model='ir.ui.view' AND module='base';
                    DELETE FROM res_lang WHERE name IN ('Serbian (Cyrillic) / српски','Belarusian / Беларусьская мова');
                    " | docker exec -i ${ODOO18_DB_HOST} psql -U ${DB_USER} -d ${ODOO18_DB} -v ON_ERROR_STOP=1

                    docker exec odoo18-web odoo \
                        -d ${ODOO18_DB} \
                        --db_host=${ODOO18_DB_HOST} \
                        --db_user=${DB_USER} \
                        --db_password=${DB_PASSWORD} \
                        --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                        --load=base,web,openupgrade_framework \
                        -u base --without-demo=all --stop-after-init --log-level=info
                '''
            }
        }

        stage('Run OpenUpgrade Migration - Remaining Modules ✅') {
            steps {
                sh '''
                    docker exec odoo18-web odoo \
                        -d ${ODOO18_DB} \
                        --db_host=${ODOO18_DB_HOST} \
                        --db_user=${DB_USER} \
                        --db_password=${DB_PASSWORD} \
                        --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                        --load=base,web,openupgrade_framework \
                        -u web,mail,account,stock,sale,purchase \
                        --without-demo=all --stop-after-init --log-level=info
                '''
            }
        }
    }

    post {
        success { echo "✅ Odoo 17 → 18 migration completed" }
        failure { echo "❌ Migration failed" }
    }
}

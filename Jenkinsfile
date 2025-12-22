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
        stage('Total Environment Cleanup 🧨') {
            steps {
                sh '''
                    echo "Cleaning EVERYTHING to free up maximum disk space..."
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

        stage('Start Odoo 17') {
            steps {
                sh 'docker compose -f docker/docker-compose-odoo17.yml up -d'
            }
        }

        stage('Wait & Init Odoo 17') {
            steps {
                sh '''
                    until docker exec ${ODOO17_DB_HOST} pg_isready -U ${DB_USER}; do sleep 5; done
                    docker exec odoo17-web odoo \
                        -d ${ODOO17_DB} \
                        -i base,sale,purchase,stock,account,mail,web \
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
                sh 'docker exec -i ${ODOO17_DB_HOST} pg_dump -U ${DB_USER} -F c ${ODOO17_DB} > ${ODOO17_DUMP}'
            }
        }

        stage('Mid-Pipeline Disk Cleanup 🧹') {
            steps {
                sh '''
                    echo "Stopping Odoo 17 and clearing space for Odoo 18 images..."
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
                        git clone --depth 1 --branch 18.0 https://github.com/OCA/OpenUpgrade.git docker/OpenUpgrade-18.0
                    fi
                    mkdir -p docker/OpenUpgrade-18.0/addons
                    rsync -a --delete docker/OpenUpgrade-18.0/openupgrade_framework/ docker/OpenUpgrade-18.0/addons/openupgrade_framework/
                '''
            }
        }

        stage('Start Odoo 18') {
            steps {
                sh 'docker compose -f docker/docker-compose-odoo18.yml up -d --remove-orphans'
            }
        }

        stage('Restore DB into Odoo 18') {
            steps {
                sh '''
                    until docker exec ${ODOO18_DB_HOST} pg_isready -U ${DB_USER}; do sleep 5; done
                    cat ${ODOO17_DUMP} | docker exec -i ${ODOO18_DB_HOST} pg_restore \
                        -U ${DB_USER} -d ${ODOO18_DB} --clean --if-exists
                '''
            }
        }

        stage('Run OpenUpgrade Migration - Base First ✅') {
            steps {
                sh '''
                    set -e
                    echo "Installing openupgradelib..."
                    docker exec -u 0 odoo18-web pip install openupgradelib --break-system-packages

                    echo "Executing Ordered View Cleanup to bypass Check Constraints..."
                    echo "
                    /* 1. Delete ALL child (extension) views that inherit from a base view */
                    DELETE FROM ir_ui_view 
                    WHERE inherit_id IN (
                        SELECT res_id FROM ir_model_data 
                        WHERE model='ir.ui.view' AND module='base'
                    );

                    /* 2. Delete the actual base views (parents) */
                    DELETE FROM ir_ui_view WHERE id IN (
                        SELECT res_id FROM ir_model_data 
                        WHERE model='ir.ui.view' AND module='base'
                    );

                    /* 3. Cleanup the XML-ID mapping */
                    DELETE FROM ir_model_data WHERE model='ir.ui.view' AND module='base';

                    /* 4. Cleanup duplicate language rows */
                    DELETE FROM res_lang WHERE name IN ('Serbian (Cyrillic) / српски','Belarusian / Беларусьская мова');
                    " | docker exec -i ${ODOO18_DB_HOST} psql -U ${DB_USER} -d ${ODOO18_DB} -v ON_ERROR_STOP=1

                    echo "Starting Base Migration..."
                    docker exec odoo18-web odoo \
                        -d ${ODOO18_DB} \
                        --db_host=${ODOO18_DB_HOST} \
                        --db_user=${DB_USER} \
                        --db_password=${DB_PASSWORD} \
                        --addons-path=/usr/lib/python3/dist-packages/odoo/addons,/mnt/openupgrade_addons \
                        --load=base,web,openupgrade_framework \
                        -u base --without-demo=all --stop-after-init
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
                        --without-demo=all --stop-after-init
                '''
            }
        }
    }

    post {
        success { echo "✅ Odoo 17 → 18 migration successful" }
        failure { echo "❌ Migration failed" }
    }
}

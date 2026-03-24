pipeline {
  agent any

  environment {
    // Swap credential ID per environment:
    // db-creds-dev | db-creds-uat | db-creds-sit
    DB_CREDS       = credentials('db-creds-dev')
    DBMATE_IMAGE   = 'ghcr.io/amacneil/dbmate:2.31.0'
    DBMATE_NETWORK = 'sandbox-net'       // update per environment
    DB_HOST        = 'sandbox-db'        // update per environment
    DB_PORT        = '3306'
    DB_NAME        = 'my_app'            // the target schema name
    DB_PARAMS      = 'tls=false&allowNativePasswords=true'
  }

  stages {

    stage('Verify version') {
      steps {
        sh 'docker run --rm ${DBMATE_IMAGE} --version'
      }
    }

    // Blocks until the database accepts connections (60s default)
    stage('Wait for DB') {
      steps {
        sh """docker run --rm \\
          --network ${DBMATE_NETWORK} \\
          -e DATABASE_URL="mysql://${DB_CREDS_USR}:${DB_CREDS_PSW}@${DB_HOST}:${DB_PORT}/${DB_NAME}?${DB_PARAMS}" \\
          ${DBMATE_IMAGE} wait"""
      }
    }

    // Applies all pending migrations in timestamp order
    stage('Migrate') {
      steps {
        sh """docker run --rm \\
          --network ${DBMATE_NETWORK} \\
          -e DATABASE_URL="mysql://${DB_CREDS_USR}:${DB_CREDS_PSW}@${DB_HOST}:${DB_PORT}/${DB_NAME}?${DB_PARAMS}" \\
          -e DBMATE_NO_DUMP_SCHEMA=true \\
          -v ${JENKINS_HOME_PATH}/workspace/${JOB_NAME}/db/migrations:/db/migrations \\
          ${DBMATE_IMAGE} migrate"""
      }
    }

    // Hard gate: exits non-zero if any migrations are still pending post-migrate
    stage('Verify') {
      steps {
        sh """docker run --rm \\
          --network ${DBMATE_NETWORK} \\
          -e DATABASE_URL="mysql://${DB_CREDS_USR}:${DB_CREDS_PSW}@${DB_HOST}:${DB_PORT}/${DB_NAME}?${DB_PARAMS}" \\
          -v ${JENKINS_HOME_PATH}/workspace/${JOB_NAME}/db/migrations:/db/migrations \\
          ${DBMATE_IMAGE} status --exit-code"""
      }
    }

  }
}

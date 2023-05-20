final SLACK_CHANNEL = '#cw-engineers'

final SLACK_COLOR = [
  ABORTED:  '#9e9e9e',  // grey
  FAILURE:  '#dc3545',  // red
  STARTING: '#0dcaf0',  // cyan
  SUCCESS:  '#198754',  // green
  UNSTABLE: '#ffc107',  // yellow
].withDefault { '#000000' }

pipeline {

  agent {
    kubernetes {
      yamlFile 'Clean_github_branch.yaml'
      defaultContainer 'autorelease'
    }
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
  }

  stages {
    stage('Query Aquents Github Organization for all Repositories') {
      steps {
        withCredentials([string(credentialsId: 'github-eisdev-token', variable: 'gh-token')]) {
          sh '''
                apt-get install -y gh
                echo $gh-token | gh auth login --with-token
                TARGET=$(date -d "+180 days" "+%Y-%m-%d")
                for repo in $(gh repo list aquent --limit 200 --json name | jq -r .[].name)
                do
                    for branch in $(gh api "/repos/aquent/$repo/branches" | jq -r .[].name)
                    do
                        commit=$(gh api "/repos/aquent/$repo/branches/$branch" --jq '.commit') 
                        date=$(jq -r '.commit | [.author.date, .committer.date] | max' <<< "$commit")
                        author=$(jq -r '[.commit.author.email] | map(select(.)) | @csv' <<< "$commit")
                        if [[ $TARGET > $date ]]
                        then
                            echo "$repo $date $branch $author" >> "branch-list.txt"
                        fi
                    done
                done
                sort -V -k 4 "branch-list.txt" > "sort-branch.txt"
             '''
        }
      }
    }
  }

  post {
    always {
      script {
        slackSend channel: SLACK_CHANNEL, color: SLACK_COLOR['SUCCESS'],
                  message: "Reminder to commit to any branches that haven't been touched in 6+ months"
        slackUploadFile channel: SLACK_CHANNEL, filePath: "sort-branch.txt"
      }
    }
  }
}

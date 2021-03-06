def git_user_email = "tobor.spins@snips.net"
def git_user_name = "Tobor"

def ssh_sh(String action) {
    sh """
     ssh-agent sh -c 'ssh-add ; $action'
    """
}

// the order of the formulae is very important: snips-platform-common must be first
def formulae = [
    "snips-platform-common.rb",
    "libsnips_megazord.rb",
    "snips-analytics.rb",
    "snips-asr-google.rb",
    "snips-asr.rb",
    "snips-audio-server.rb",
    "snips-dialogue.rb",
    "snips-hotword.rb",
    "snips-injection.rb",
    "snips-nlu.rb",
    "snips-skill-server.rb",
    "snips-tts.rb",
    "snips-watch.rb",
]

node("macos-elcapitan-homebrew") {

    def platformTag = "${params.tag}"
    def formulaPaths = formulae.collect { formula -> "Formula/${formula}" }.join(" ")
    def formulaPathsToAudit = formulae.findAll{ it != "libsnips_megazord.rb" }.collect { formula -> "Formula/${formula}" }.join(" ")
    def git_url = "git@github.com:snipsco/snips-platform.git"

    properties([
        parameters([
            string(defaultValue: 'NONE', description: 'tag to build', name: 'tag'),
            booleanParam(defaultValue: false, description: 'merge,upload and push to prod', name: 'push_to_prod'),
        ])
    ])

    stage('Checkout') {
        deleteDir()
        checkout scm
    }

    stage('Audit Formula') {
       ssh_sh """
            set -e

            git config --global user.email ${git_user_email}
            git config --global user.name ${git_user_name}

            git clone --branch $platformTag --depth 1 ${git_url}
            revision=\$(cd snips-platform && git rev-parse $platformTag)

            .ci/bump.sh $platformTag \$revision $formulaPaths

            .ci/audit.sh $formulaPathsToAudit
       """
    }
    
    formulae.each{ formula ->
        def formulaPath = "Formula/${formula}"
        stage("Make ${formula}") {
            ssh_sh """
                set -e
                .ci/make_bottles.sh $formulaPath
            """
        }
    }

    if(params.push_to_prod.toBoolean()) {
        stage('Release') {

            ssh_sh """
                 set -e

                 git config --global user.email ${git_user_email}
                 git config --global user.name ${git_user_name}

                 .ci/rename_bottles.sh '*.bottle.json'

                 .ci/merge.sh '*.bottle.json'

                 .ci/upload_bottles.sh

                 git checkout master
                 git commit -am "[Release] ${platformTag}"
                 git push origin master
            """
        }
    }
}


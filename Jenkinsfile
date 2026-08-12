@Library("acp3_shared_library") _


import CredentialManager.*
import SoftwareClusterFilesystemGenerator.*
import ImageCreationEnvironmentSetUp.*
import SoftwareClusterImageGenerator.*
import SoCImageDownloader.*
import SoCDevPackageAssembler.*
import SetInputConfigMap.*
import SoftwareClusterFilesystemInjector.*
import ProdPackageInputGenerator.*
import DetermineSpuSocVersion.*
import ProdPackageGenerator.DeltaPackageGenerator.*
import ProdPackageGenerator.FullPackageGenerator.*
import ProdPackageGenerator.ReverseDeltaPackageGenerator.*
import UCMDbGenerator.*
import CalProdPackager.*

env.unique_id = UUID.randomUUID().toString()
env.SWCL_ROOT_DIR = 'sw_cluster_contents'
env.IMAGE_OUTPUT_PATH = 'SoCImagesToUpload'
env.QTD_IMAGE_OUTPUT_PATH = 'SoCQTDImagesToUpload'
env.IMAGE_CREATION_ENV_USER = 'ACPx_Prebuilds'
env.conan_pkg_settings = "-s compiler=gcc -s compiler.version=7 -s compiler.libcxx=libstdc++"
env.PACK_PATH = ''
//Platform team : shared_lib is the label used in CTC, please use the labels given to your agents
env.agent_label = 'shared_lib'
//Platform team : The below ENV vars are provided by platform team
env.GM_QNX_CHANNEL = 'profile'
env.node_name
env.current_multi_img_dir = "current_multi_images"
env.old_multi_img_dir = "old_multi_images"
env.multi_swcl_gen_ws = "swcl_generation_ws"
env.multi_delta_swcl_gen_ws = "swcl_generation_delta_ws"
env.swcl_gen_sub_dir = "vector_manifests"
env.shipping_pkg_dir = "shipping_pkg_dir"
env.current_qtd_img_staging_dir = "current_qtd_img_staging"
env.artifactory_url = "https://artifactory-wrn.ultracruise.gm.com/artifactory"
env.version_tracking_db_location = "acp3-archive-local/ASCII_version_database/"
env.DEV_PACK_PATH = ''
env.PROD_PACK_NAME = ''
env.SOC0_JSON_NAME = ''
//The pipeline will generate and upload 1, cluster filesystem zip file
//                                      2, cluster image conan dev pkg with hash value to differentiate
//                                      3, cluster image conan prod pkg with hash value to differentiate
//                                      4, qtd prebuild pkg (qtd_use)
//                                      5, QDownloadable package as tar.gz
//                                      6, QDownloadable package as conan package
properties([
  parameters([
    separator(name: "BUILD_ENVIRONMENT AND CONTROLLER SELECTION", sectionHeader: "Environment and Controller",
        separatorStyle: "border-width: 0",
        sectionHeaderStyle: """
            background-color: #7ea6d3;
            text-align: center;
            padding: 4px;
            color: #343434;
            font-size: 22px;
            font-weight: normal;
            text-transform: uppercase;
            font-family: 'Orienta', sans-serif;
            letter-spacing: 1px;
            font-style: italic;
        """
    ),
    hidden(
      name: 'AgentLabel',
      defaultValue: 'shared_lib',
      description: 'Hidden Param for Platform Release Pipeline this should always be set to shared_lib - This decide the agent for running the core imagecreation routine. Example labels: 1)shared_lib , 2)ubuntu_18, 3)test_agent'
    ),
    choice(
      name: 'TARGET',
      choices: ['ACP3','ACP3.1'],
      description: 'Required* Target Controller for which images are being generated'
    ),
    separator(name: "IMAGE CREATION PROPERTIES", sectionHeader: "Image Creation Properties",
        separatorStyle: "border-width: 0",
        sectionHeaderStyle: """
            background-color: #7ea6d3;
            text-align: center;
            padding: 4px;
            color: #343434;
            font-size: 22px;
            font-weight: normal;
            text-transform: uppercase;
            font-family: 'Orienta', sans-serif;
            letter-spacing: 1px;
            font-style: italic;
        """
    ),
    booleanParam(
    name: 'skip_image_creation',
    defaultValue: false,
    description: 'Do not use, meant for testing purpose - Skips image creation and can do Dev and Prod packaging on hardcoded cluster versions.'
    ),
    [$class: 'CascadeChoiceParameter',
      choiceType: 'PT_CHECKBOX',
      description: 'If needed - Select app in Application cluster to set its log_level to kDebug',
      filterLength: 1,
      filterable: false,
      name: 'log_level_application',
      referencedParameters: '',
      script:
      [$class: 'GroovyScript',
        fallbackScript: [
          classpath: [],
          sandbox: true,
          script: "return ['']"
        ],
        script: [
          classpath: [],
          sandbox: true,
          script: '''
              return [
                'ASUM_SPU',
                'ASUM_Diag',
                'DMS_Manager'
              ]
          '''
        ]
      ]
    ],
    [$class: 'CascadeChoiceParameter',
      choiceType: 'PT_CHECKBOX',
      description: 'If needed - Select app in AdaptiveExecutables cluster to set its log_level to kDebug',
      filterLength: 1,
      filterable: false,
      name: 'log_level_adaptive',
      referencedParameters: '',
      script:
      [$class: 'GroovyScript',
        fallbackScript: [
          classpath: [],
          sandbox: true,
          script: "return ['']"
        ],
        script: [
          classpath: [],
          sandbox: true,
          script: '''
              return [
                'amsr_vector_fs_updatemanager',
                'ASUM_SEC',
                'DiagDaemonExe'
              ]
          '''
        ]
      ]
    ],
    booleanParam(
          name: 'merge_existing_apps',
          defaultValue: false,
          description: 'If needed - Inject Additional Software Cluster content'
        ),
    [$class: 'DynamicReferenceParameter',
      choiceType: 'ET_FORMATTED_HTML',
      omitValueField: true,
      description: 'If needed - Version of the first Application SW Cluster on SOC',
      name: 'Application',
      referencedParameters: 'merge_existing_apps',
      script: [
              $class: 'GroovyScript',
              fallbackScript: [
                      classpath: [],
                      sandbox: true,
                      script:
                              'return[\'nothing.....\']'
              ],
              script: [
                      classpath: [],
                      sandbox: true,
                      script:
                          """

                              if(merge_existing_apps) {
                                inputBox="<input name='value' type='text' value=''>"
                              } else {
                                inputBox="<input name='value' type='text' value='' disabled>"
                              }
                          """
              ]
      ]
    ],
    [$class: 'DynamicReferenceParameter',
      choiceType: 'ET_FORMATTED_HTML',
      omitValueField: true,
      description: 'If needed - Version of the AdaptiveExecutables SW Cluster on SOC',
      name: 'AdaptiveExecutables',
      referencedParameters: 'merge_existing_apps',
      script: [
              $class: 'GroovyScript',
              fallbackScript: [
                      classpath: [],
                      sandbox: true,
                      script:
                              'return[\'nothing.....\']'
              ],
              script: [
                      classpath: [],
                      sandbox: true,
                      script:
                          """

                              if(merge_existing_apps) {
                                inputBox="<input name='value' type='text' value=''>"
                              } else {
                                inputBox="<input name='value' type='text' value='' disabled>"
                              }
                          """
              ]
      ]
    ],
    [$class: 'DynamicReferenceParameter',
      choiceType: 'ET_FORMATTED_HTML',
      omitValueField: true,
      description: 'If needed - Version of the Calibration SW Cluster on SOC',
      name: 'Calibration',
      referencedParameters: 'merge_existing_apps',
      script: [
              $class: 'GroovyScript',
              fallbackScript: [
                      classpath: [],
                      sandbox: true,
                      script:
                              'return[\'nothing.....\']'
              ],
              script: [
                      classpath: [],
                      sandbox: true,
                      script:
                          """
                              if(merge_existing_apps) {
                                inputBox="<input name='value' type='text' value=''>"
                              } else {
                                inputBox="<input name='value' type='text' value='' disabled>"
                              }
                          """
              ]
      ]
    ],
    [$class: 'DynamicReferenceParameter',
      choiceType: 'ET_FORMATTED_HTML',
      omitValueField: true,
      description: 'If needed - Version of the NN_Models SW Cluster on SOC',
      name: 'NN_Models',
      referencedParameters: 'merge_existing_apps',
      script: [
              $class: 'GroovyScript',
              fallbackScript: [
                      classpath: [],
                      sandbox: true,
                      script:
                              'return[\'nothing.....\']'
              ],
              script: [
                      classpath: [],
                      sandbox: true,
                      script:
                          """

                              if(merge_existing_apps) {
                                inputBox="<input name='value' type='text' value=''>"
                              } else {
                                inputBox="<input name='value' type='text' value='' disabled>"
                              }
                          """
              ]
      ]
    ],
    separator(name: "Input Configurations", sectionHeader: "Input Configurations",
        separatorStyle: "border-width: 0",
        sectionHeaderStyle: """
            background-color: #7ea6d3;
            text-align: center;
            padding: 4px;
            color: #343434;
            font-size: 22px;
            font-weight: normal;
            text-transform: uppercase;
            font-family: 'Orienta', sans-serif;
            letter-spacing: 1px;
            font-style: italic;
        """
    ),
    [$class: 'DynamicReferenceParameter',
      choiceType: 'ET_FORMATTED_HTML',
      omitValueField: true,
      description: 'Required* The software packages: used to 1,differentiate the artifacts by uploading to unique locations; 2,generate calibration schema for building Application and/or Calibration',
      name: 'soc_version',
      // randomName: 'choice-parameter-5631314456178624',
      referencedParameters: '',
      script: [
              $class: 'GroovyScript',
              fallbackScript: [
                      classpath: [],
                      sandbox: true,
                      script:
                              'return[\'nothing.....\']'
              ],
              script: [
                      classpath: [],
                      sandbox: true,
                      script:
                          """
                          inputBox="<input name='value' type='text' value='Rx.y.x'>"

                          """
              ]
      ]
    ],
    string(
      name: 'acpx_soc_buildrelease_branch',
      defaultValue: 'master',
      trim: true,
      description: 'Autoselected, No action needed - Branch to clone from acpx_soc_buildrelease repo'
    ),
    choice(
        name: 'PKG_STR_NAME_PLATFORM',
        choices: ['platform', 'platform-apps', 'platform-fusion'],
        description: 'Required* Platform conan package name'
    ),
    string(
      name: 'PKG_VER_PLATFORM',
      trim: true,
      description: 'Required* Platform conan package version'
    ),
    string(
      name: 'PKG_VER_ADAS_SOC',
      trim: true,
      description: 'Optional* ADAS SoC package version'
    ),
    string(
      name: 'ADDITIONAL_ENV_VARIABLES',
      trim: true,
      description: 'Additional environment variables as a comma-separated list. example: TEST1=data1,TEST2=data2'
    ),
    string(
      name: 'GM_QNX_USER',
      defaultValue: 'qnx.7.1.0.QOS221_20220922_ASLR.llvm',
      trim: true,
      description: 'Required* QNX OS Conan user'
    ),
    [$class: 'DynamicReferenceParameter',
      choiceType: 'ET_FORMATTED_HIDDEN_HTML',
      omitValueField: true,
      description: '''REBUILD_OPTION - THIS NEEDS TO BE MANUALLY SET IF CHANGED - This is the program Model_Year.''',
      filterLength: 1,
      filterable: false,
      name: 'Model_Year',
      referencedParameters: 'PKG_VER_ADAS_SOC',
      script:
      [$class: 'GroovyScript',
        fallbackScript: [
          classpath: [],
          sandbox: true,
          script: 'return[\'nothing.....\']'
        ],
        script: [
          classpath: [],
          sandbox: true,
          script: '''
            if(PKG_VER_ADAS_SOC.contains(".")) {
              inputBox="<input name='value' type='text' value='8888' disabled>"
            } else {
              inputBox="<input name='value' type='text' value='9999' disabled>"
            }
          '''
        ]
      ]
    ],
    separator(name: "QDownload Only", sectionHeader: "QDownload Only",
        separatorStyle: "border-width: 0",
        sectionHeaderStyle: """
            background-color: #7ea6d3;
            text-align: center;
            padding: 4px;
            color: #343434;
            font-size: 22px;
            font-weight: normal;
            text-transform: uppercase;
            font-family: 'Orienta', sans-serif;
            letter-spacing: 1px;
            font-style: italic;
        """
    ),
    booleanParam(
    name: 'skip_qdownload_generation',
    defaultValue: false,
    description: 'If needed - Skip QDownload generation.'
    ),
    string(
      name: 'SoC_Image_suffix_uid',
      defaultValue: '',
      description: '''If needed - Optional prefix to QDownload filename - \n
      SoC_Image_<SoC_Version>_<bsp_version>_<package_purpose>_<boot_security>_<env.ufs_size>_<SoC_Image_suffix_uid>. \n
      If this filename is the same as a previous build, it will overwrite the older one with the same filename.''',
      trim: true
    ),
    separator(name: "Common for QDownload and OTA Packages", sectionHeader: "Common for QDownload and OTA Packages",
        separatorStyle: "border-width: 0",
        sectionHeaderStyle: """
            background-color: #7ea6d3;
            text-align: center;
            padding: 4px;
            color: #343434;
            font-size: 22px;
            font-weight: normal;
            text-transform: uppercase;
            font-family: 'Orienta', sans-serif;
            letter-spacing: 1px;
            font-style: italic;
        """
    ),
    choice(
      name: 'package_purpose',
      choices: ['',
      'development',
      'production'],
      description: 'Required* Selecting package purpose for SOC Image package creation to pull BSP packages'
    ),
    choice(
      name: 'boot_security',
      choices: ['',
      'nonsecure',
      'secure'],
      description: 'Required* Selecting boot security for SOC Image package creation to pull BSP packages'
    ),
    choice(
      name: 'qtd_use',
      choices: ['', 'qtd', 'noqtd'],
      description: 'Required* Selecting mod-secure type (qtd/noqtd) for SOC Image package creation to pull BSP packages'
    ),
    hidden(
      name: 'devops_tool_branch',
      defaultValue: 'master',
      description: 'Branch to clone from devops_tool repo'
    ),
    separator(name: "OTA Specific", sectionHeader: "OTA Specific",
        separatorStyle: "border-width: 0",
        sectionHeaderStyle: """
            background-color: #7ea6d3;
            text-align: center;
            padding: 4px;
            color: #343434;
            font-size: 22px;
            font-weight: normal;
            text-transform: uppercase;
            font-family: 'Orienta', sans-serif;
            letter-spacing: 1px;
            font-style: italic;
        """
    ),
    booleanParam(
    name: 'trigger_prod_packager',
    defaultValue: false,
    description: 'If needed - Trigger OTA package creation.'
    ),
    [$class: 'CascadeChoiceParameter',
     choiceType: 'PT_CHECKBOX',
     description: 'Select Check box or type True on rebuild, if full package is expected. Generate full package for the required clusters (with Action Type="Update")',
     name: 'full_package',
     referencedParameters: 'trigger_prod_packager',
     script: [
         $class: 'GroovyScript',
         script: [
             classpath: [],
             sandbox: true,
             script: '''
                 if (trigger_prod_packager) {
                     return ['True']
                 } else {
                     return ['Generate full package:disabled']
                 }
             '''
         ],
         fallbackScript: [
             classpath: [],
             sandbox: true,
             script: "return ['Script Error, contact pipeline owners']"
         ]
     ],
    ],
    [$class: 'CascadeChoiceParameter',
     choiceType: 'PT_CHECKBOX',
     description: 'Select Check box or type True on rebuild, if Cal package is expected. Generate Calibration package',
     name: 'cal_package',
     referencedParameters: 'trigger_prod_packager',
     script: [
         $class: 'GroovyScript',
         script: [
             classpath: [],
             sandbox: true,
             script: '''
                 if (trigger_prod_packager) {
                     return ['True']
                 } else {
                     return ['Generate Cal package:disabled']
                 }
             '''
         ],
         fallbackScript: [
             classpath: [],
             sandbox: true,
             script: "return ['Script Error, contact pipeline owners']"
         ]
     ]
    ],
    [$class: 'CascadeChoiceParameter',
     choiceType: 'PT_CHECKBOX',
     description: 'Select Check box or type True on rebuild, if delta package is expected. Generate delta package for the required clusters (with Action Type="Update")',
     name: 'delta_package',
     referencedParameters: 'trigger_prod_packager',
     script: [
         $class: 'GroovyScript',
         script: [
             classpath: [],
             sandbox: true,
             script: '''
                 if (trigger_prod_packager) {
                     return ['True']
                 } else {
                     return ['Generate delta package:disabled']
                 }
             '''
         ],
         fallbackScript: [
             classpath: [],
             sandbox: true,
             script: "return ['Script Error, contact pipeline owners']"
         ]
     ],
    ],
    string(
      name: 'delta_from_manifest_link',
      defaultValue: '',
      description: 'Optional* Provide the link to the delta from package manifest, if requesting delta or reverse_delta package'
    ),
    [$class: 'CascadeChoiceParameter',
     choiceType: 'PT_CHECKBOX',
     description: 'Select Check box or type True on rebuild, if reverse delta package is expected. Generate reverse delta package for the required clusters (with Action Type="Update")',
     name: 'reverse_delta_package',
     referencedParameters: 'trigger_prod_packager',
     script: [
         $class: 'GroovyScript',
         script: [
             classpath: [],
             sandbox: true,
             script: '''
                 if (trigger_prod_packager) {
                     return ['True']
                 } else {
                     return ['Generate reverse delta package:disabled']
                 }
             '''
         ],
         fallbackScript: [
             classpath: [],
             sandbox: true,
             script: "return ['Script Error, contact pipeline owners']"
         ]
     ],
    ],
    [$class: 'CascadeChoiceParameter',
     choiceType: 'PT_CHECKBOX',
     description: 'Autoselected, no action needed. Generate shipping_image',
     name: 'shipping_image',
     referencedParameters: 'trigger_prod_packager',
     script: [
         $class: 'GroovyScript',
         script: [
             classpath: [],
             sandbox: true,
             script: '''
                 if (trigger_prod_packager) {
                     return ['False:disabled']
                 } else {
                     return ['Generate shipping_image:disabled']
                 }
             '''
         ],
         fallbackScript: [
             classpath: [],
             sandbox: true,
             script: "return ['Script Error, contact pipeline owners']"
         ]
     ]
    ],
    hidden(
     name: 'ASCII_Version',
     defaultValue: '11',
     description: 'Hidden Param - This is the SPU SoC Version'
    ),
    hidden(
      name: 'CalSWCL',
      defaultValue: 'ACP3_Calibration',
      description: 'Hidden Param - Passing hardcoded default value to CalProdPackager'
    ),
    hidden(
      name: 'envelope_sign',
      defaultValue: 'true',
      description: 'Hidden Param - Passing hardcoded default value to CalProdPackager'
    ),
    hidden(
        name: 'create_tag',
        defaultValue: 'false',
        description: 'Create tag in CAL_ACP.acpx_soc_calibration repo, Example: 00000000AA'
    ),
    hidden(
      name: 'Cal_source',
      defaultValue: 'manifestjson',
      description: 'Hidden Param - Passing hardcoded default value to CalProdPackager'
    ),
    hidden(
      name: 'acpx_soc_calibration_branch',
      defaultValue: 'soc_version',
      description: 'Hidden Param - Passing hardcoded default value to CalProdPackager'
    ),
    [$class: 'CascadeChoiceParameter',
     choiceType: 'PT_CHECKBOX',
     description: 'Auto selected, no action needed.  - When enabled, will update the SoC to ASCII version database unless value exists already',
     name: 'parse_SPU_version_tracking_db',
     referencedParameters: 'trigger_prod_packager',
     script: [
         $class: 'GroovyScript',
         script: [
             classpath: [],
             sandbox: true,
             script: '''
                 if (trigger_prod_packager) {
                     return ['Parse SPU-SOC version tracking Db:disabled']
                 } else {
                     return ['Parse SPU-SOC version tracking Db:disabled']
                 }
             '''
         ],
         fallbackScript: [
             classpath: [],
             sandbox: true,
             script: "return ['Script Error, contact pipeline owners']"
         ]
     ],
    ],
    separator(name: "OTHER", sectionHeader: "Other",
        separatorStyle: "border-width: 0",
        sectionHeaderStyle: """
            background-color: #7ea6d3;
            text-align: center;
            padding: 4px;
            color: #343434;
            font-size: 22px;
            font-weight: normal;
            text-transform: uppercase;
            font-family: 'Orienta', sans-serif;
            letter-spacing: 1px;
            font-style: italic;
        """
    ),
    choice(
     name: 'artifactory_upload_location_root',
     choices: ['acpx-soc-release/testing', 'acpx-soc-test'],
     description: 'Change if needed - Root path for Artifactory uploads.'
    ),
    booleanParam(
      name: 'force_override_package',
      defaultValue: 'false',
      description: 'Use with caution - Select only if you wish to override an existing package on artifactory with the newly generated one.'
    )
  ])
])
pipeline {
    agent none
    options {
        timeout(time: 6, unit: 'HOURS')
        buildDiscarder logRotator(
      artifactDaysToKeepStr: '',
      artifactNumToKeepStr: '',
      daysToKeepStr: '30',
      numToKeepStr: '20'
    )
        preserveStashes(buildCount: 5)
        parallelsAlwaysFailFast()
    }
    stages {
        stage('Pipeline Setup') {
            agent { label 'docker' }
            steps {
                script {
                    env.artifactory_ui_url = new CredentialManager(
            this.steps,
            CredentialManager.DevZoneService.E_ARTIFACTORY_WRN
          ).GetServiceUrl()
                }
            }
        }

        stage('Get Filesystem Configuration for platform') {
            agent { label env.agent_label }
            steps {
                script {
                    new SetInputConfigMap(this).RunRoutine()
                }
            }
    }//Get Filesystem Configuration for paltform

        stage('Parallel Steps') {
            parallel {
                stage('Software Cluster Filesystem Generator') {
                    when {
                        expression { return !("$skip_image_creation" ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/) }
                    }
                    agent { label env.agent_label }
                    steps {
                        script {
                            new SoftwareClusterFilesystemGenerator(this).RunRoutine()
                        }
                    }
        }//Software Cluster Filesystem Generator

                stage('Prebuild environment preparation') {
                    when {
                        expression { return !("$skip_image_creation" ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/) }
                    }
                    agent { label params.AgentLabel }
                    steps {
                        script {
                            // Store current node name in an env var to allow subsequent stages can run on the same agent
                            env.node_name = "${NODE_NAME}"
                            new ImageCreationEnvironmentSetUp(this).RunRoutine()
                        }
                    }
        }//Prebuild environment preparation
      }//parallel
    }//Parallel Steps
        stage('Inject Software Cluster File systems') {
            when {
                expression { return ("$merge_existing_apps" ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/) && !("$skip_image_creation" ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/) }
            }
            agent { label params.AgentLabel }
            steps {
                script {
                    new SoftwareClusterFilesystemInjector(this).RunRoutine()
                }
            }
    }//Inject Software Cluster Image(s)
        stage('UCM DB update') {
            when {
            expression { return !("$skip_image_creation" ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/) }
          }
          agent {
                label env.agent_label
            }
            steps {
                script {
                    new UCMDbGenerator(this).RunRoutine()
                }
            }
    }//UCM DB update
        stage('Create the Software Cluster Image(s)') {
            when {
                expression { return !("$skip_image_creation" ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/) }
            }
            agent {
                label env.node_name
            }
            steps {
                script {
                    new SoftwareClusterImageGenerator(this).RunRoutine()
                }
            }
    }//Create the Software Cluster Image(s)

        stage('Prepare inputs and generate development package(s)') {
            agent { label params.AgentLabel }
            when {
                expression { return !("$skip_qdownload_generation" ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/) }
            }
            stages {
                stage('Download SoC Cluster Images from artifactory') {
                    steps {
                        script {
                            new SoCImageDownloader(this).RunRoutine()
                        }
                    }
        }//Download SoC Cluster Images from artifactory
                stage('Assemble SoC Dev package from the downloaded images') {
                    steps {
                        script {
                            new SoCDevPackageAssembler(this).RunRoutine()
                        }
                    }
        }//Assemble SoC Dev package from the downloaded images
            }
        }
        stage('Prepare inputs and generate production package(s)') {
            when {
                expression { return "$trigger_prod_packager" ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/ }
            }
            agent {
                label env.agent_label
            }
            stages {
                stage('Prepare the inputs for production package generation') {
                    steps {
                        script {
                            new ProdPackageInputGenerator(this).RunRoutine()
                        }
                    }
        }//Prepare inputs and generate production package(s)
                stage('Determine Applicable Software Part Update (SPU) SoC Version in ASCII format') {
                    steps {
                        script {
                            new DetermineSpuSocVersion(this).RunRoutine()
                        }
                    }
        }//Determine Applicable Software Part Update (SPU) SoC Version in ASCII format
                stage('Parallel Steps-creating packages') {
                    parallel {
                        stage('Create Full Production Package') {
                            agent { label env.agent_label }
                            when {
                                expression { params.full_package.toBoolean() == true }
                            }
                            steps {
                                script {
                                    new FullPackageGenerator(this).RunRoutine()
                                }
                            }
            }//Create Full Production Package
                        stage('Create Delta Production Package') {
                            agent { label env.agent_label }
                            when {
                                expression { params.delta_package.toBoolean() == true }
                            }
                            steps {
                                script {
                                    new DeltaPackageGenerator(this).RunRoutine()
                                }
                            }
            }//Create Delta Production Package
                        stage('Create Reverse Delta Production Package') {
                            agent { label env.agent_label }
                            when {
                                expression { params.reverse_delta_package.toBoolean() == true }
                            }
                            steps {
                                script {
                                    new ReverseDeltaPackageGenerator(this).RunRoutine()
                                }
                            }
            }//Create Reverse Delta Production Package
            stage('Create Calibration Package') {
                agent { label env.agent_label }
                    when {
                expression {params.cal_package ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/ }
              }
              steps {
                                script {
                                    new CalProdPackager(this).RunRoutine()
                                }
                            }
            }//Create Calibration Package
                    }
                }
            }
        }
    }
}
# samplebuild



name: 2.0.$(Build.BuildId)
trigger:
  branches:
    include:
      - master

variables:
  - template: variablesdefinition.yml
  - group: onlineShopping.UI
  - group: onlineShopping.UI.secrets
  - group: changemanagement.lib.prod
  - group: changemanagement.online.lib.prod

resources:
  pipelines:
    - pipeline: 'OnlineShopping.UI.Containerised'
      project: OnlineShopping
      source: 'OnlineShopping.UI.Containerised'
      branch: master
      trigger:
        branches:
          - master

  repositories:
    - repository: 'OnlineShopping.UI'
      project: OnlineShopping
      name: 'OnlineShopping.UI'
      type: git

  containers:
    - container: cypress-test-agent
      image: wwnzregistrydev.azurecr.io/trader/test/cypress/included:7.4.0
      endpoint: WWNZDEVTESTREGISTRY

pool:
  name: $(TraderPoolName)

stages:
  - template: build/build.yml
    parameters:
      stageName: 'Build_Test'
      websiteName: $(WebsiteNameDev)
      resourceGroupName: $(ResourceGroupNameDev)
      subscription: $(SubscriptionDev)

  - template: release/storybook.yml
    parameters:
      stageName: 'storybook'
      stageDependency: 'Build_Test'
      targetStorage: 'wwnztraderprodstatic'
      subscription: $(SubscriptionProd)

  - template: release/package-and-build-image.yml
    parameters:
      stageName: 'packageBuildImage'
      stageDependency: 'Build_Test'
      websiteName: $(WebsiteNameDev)
      resourceGroupName: $(ResourceGroupNameDev)
      subscription: $(SubscriptionDev)
      displayName: 'Package and Build Image'

  - template: release/ui-asset-to-blob.yml
    parameters:
      stageName: 'uiassettoblob'
      stageDependency: 'Build_Test'
      deployment: 'uiassetscopy'
      displayName: 'Copy UI Assets to Azure blob'
      environment: 'OnlineShoppingUIAssets'
      subscription: $(SubscriptionProd)
      targetStorage: 'wwnztraderprodstatic'
      targetcontainer: 'assets'

  - template: release/tag-branch.yml
    parameters:
      stageName: 'tagMasterbranch'
      stageDependency: 'uiassettoblob'
      deployment: 'tagTargetMasterBranch'
      displayName: 'Tag master with current BuildNumber'
      environment: 'TagMasterBranch'
      buildnumber: $(Build.BuildNumber)

  - template: release/appservice-slot-deploy.yml
    parameters:
      stageName: 'ci'
      stageDependency: [uiassettoblob, packageBuildImage]
      websiteName: $(WebsiteNameDev)
      websiteNameSSR: $(WebsiteNameDevSSR)
      resourceGroupName: $(ResourceGroupNameDev)
      subscription: $(SubscriptionDev)
      deployment: 'cideployment'
      displayName: 'CI Deployment'
      environment: 'OnlineShoppingUICI'
      GIGYA_CUSTOM_DOMAIN: $(GIGYA_CUSTOM_DOMAIN)
      API_PORT: $(API_PORT)
      API_URL: $(API_URL_DEV)
      APPINSIGHTS_KEY: $(APPINSIGHTS_KEY_DEV)
      APPINSIGHTS_INSTRUMENTATIONKEY: $(APPINSIGHTS_KEY_DEV)
      APPINSIGHTS_SSR_NODENAME: $(APPINSIGHTS_SSR_NODENAME)
      APPINSIGHTS_LIVE: $(APPINSIGHTS_LIVE_DEV)
      DOCKER_CUSTOM_IMAGE_NAME: $(DOCKER_CUSTOM_IMAGE_NAME_DEV)
      DOCKER_SSR_CUSTOM_IMAGE_NAME: $(DOCKER_SSR_CUSTOM_IMAGE_NAME_DEV)
      DOCKER_REGISTRY_SERVER_PASSWORD: $(DOCKER_REGISTRY_SERVER_PASSWORD_DEV)
      DOCKER_REGISTRY_SERVER_URL: $(DOCKER_REGISTRY_SERVER_URL_DEV)
      DOCKER_REGISTRY_SERVER_USERNAME: $(DOCKER_REGISTRY_SERVER_USERNAME_DEV)
      GIGYA_API_KEY: $(GIGYA_API_KEY_DEV)
      GTM_AUTH: $(GTM_AUTH_DEV)
      SITE_LOCATION_API_URL: $(SITE_LOCATION_API_URL_DEV)
      slotName: production
      ssrSlotName: production
      deployToSlotOrASE: true
      dockerNamespace: $(DevDockerRegistry)
      oliveChatUrl: $(OLIVE_CHAT_WIDGET_URL_DEV)
      citrus_ad_base_url: $(CITRUS_AD_BASE_URL_DEV)

  - template: release/e2e-test.yml
    parameters:
      stageName: 'e2etest'
      stageDependency: 'ci'

  - template: release/appservice-slot-deploy.yml
    parameters:
      stageName: 'qa'
      stageDependency: 'e2etest'
      websiteName: $(WebsiteNameQa)
      websiteNameSSR: $(WebsiteNameQaSSR)
      resourceGroupName: $(ResourceGroupNameQa)
      subscription: $(SubscriptionQa)
      deployment: 'qadeployment'
      displayName: 'QA Deployment'
      environment: 'OnlineShoppingUIQA'
      GIGYA_CUSTOM_DOMAIN: $(GIGYA_CUSTOM_DOMAIN)
      API_PORT: $(API_PORT)
      API_URL: $(API_URL_QA)
      APPINSIGHTS_KEY: $(APPINSIGHTS_KEY_QA)
      APPINSIGHTS_INSTRUMENTATIONKEY: $(APPINSIGHTS_KEY_QA)
      APPINSIGHTS_SSR_NODENAME: $(APPINSIGHTS_SSR_NODENAME)
      APPINSIGHTS_LIVE: $(APPINSIGHTS_LIVE_QA)
      DOCKER_CUSTOM_IMAGE_NAME: $(DOCKER_CUSTOM_IMAGE_NAME_DEV)
      DOCKER_SSR_CUSTOM_IMAGE_NAME: $(DOCKER_SSR_CUSTOM_IMAGE_NAME_DEV)
      DOCKER_REGISTRY_SERVER_PASSWORD: $(DOCKER_REGISTRY_SERVER_PASSWORD_DEV)
      DOCKER_REGISTRY_SERVER_URL: $(DOCKER_REGISTRY_SERVER_URL_DEV)
      DOCKER_REGISTRY_SERVER_USERNAME: $(DOCKER_REGISTRY_SERVER_USERNAME_DEV)
      GIGYA_API_KEY: $(GIGYA_API_KEY_QA)
      GTM_AUTH: $(GTM_AUTH_QA)
      SITE_LOCATION_API_URL: $(SITE_LOCATION_API_URL_QA)
      forceContinuation: false
      slotName: production
      ssrSlotName: production
      deployToSlotOrASE: true
      dockerNamespace: $(DevDockerRegistry)
      oliveChatUrl: $(OLIVE_CHAT_WIDGET_URL_QA)
      citrus_ad_base_url: $(CITRUS_AD_BASE_URL_QA)
      CDXQA_ACCESS_TOKEN: $(CDXQA_ACCESS_TOKEN_QA)

  - template: release/copy-prod-registry.yml
    parameters:
      stageName: 'copyprodregistry'
      stageDependency: 'qa'
      forceContinuation: false

  - template: release/manualUserInput.yml
    parameters:
      stageName: 'teamManualInterventionProgress'
      stageDependency: 'copyprodregistry'
      forceContinuation: false
      environment: 'OnlineShoppingUITeamIntervention'

  - template: release/appservice-slot-deploy.yml
    parameters:
      stageName: 'uat'
      stageDependency: 'teamManualInterventionProgress'
      websiteName: $(WebsiteNameUat)
      websiteNameSSR: $(WebsiteNameUatSSR)
      resourceGroupName: $(ResourceGroupNameUat)
      subscription: $(SubscriptionUat)
      deployment: 'uatdeployment'
      displayName: 'UAT Deployment'
      environment: 'OnlineShoppingUIUAT'
      GIGYA_CUSTOM_DOMAIN: $(GIGYA_CUSTOM_DOMAIN)
      API_PORT: $(API_PORT)
      API_URL: $(API_URL_UAT)
      APPINSIGHTS_KEY: $(APPINSIGHTS_KEY_UAT)
      APPINSIGHTS_INSTRUMENTATIONKEY: $(APPINSIGHTS_KEY_UAT)
      APPINSIGHTS_SSR_NODENAME: $(APPINSIGHTS_SSR_NODENAME)
      APPINSIGHTS_LIVE: $(APPINSIGHTS_LIVE_UAT)
      DOCKER_CUSTOM_IMAGE_NAME: $(DOCKER_CUSTOM_IMAGE_NAME_PROD)
      DOCKER_SSR_CUSTOM_IMAGE_NAME: $(DOCKER_SSR_CUSTOM_IMAGE_NAME_PROD)
      DOCKER_REGISTRY_SERVER_PASSWORD: $(DOCKER_REGISTRY_SERVER_PASSWORD_PROD)
      DOCKER_REGISTRY_SERVER_URL: $(DOCKER_REGISTRY_SERVER_URL_PROD)
      DOCKER_REGISTRY_SERVER_USERNAME: $(DOCKER_REGISTRY_SERVER_USERNAME_PROD)
      GIGYA_API_KEY: $(GIGYA_API_KEY_UAT)
      GTM_AUTH: $(GTM_AUTH_UAT)
      SITE_LOCATION_API_URL: $(SITE_LOCATION_API_URL_UAT)
      forceContinuation: false
      slotName: production
      ssrSlotName: production
      deployToSlotOrASE: true
      dockerNamespace: $(ProdDockerRegistry)
      oliveChatUrl: $(OLIVE_CHAT_WIDGET_URL_UAT)
      citrus_ad_base_url: $(CITRUS_AD_BASE_URL_UAT)
      CDXQA_ACCESS_TOKEN: $(CDXQA_ACCESS_TOKEN_UAT)

  ###########################################################
  ## PROD
  ###########################################################

  - template: release/appservice-slot-deploy.yml
    parameters:
      stageName: 'preprodslot'
      stageDependency: 'uat'
      websiteName: $(WebsiteNameProd)
      websiteNameSSR: $(WebsiteNameProdSSR)
      resourceGroupName: $(ResourceGroupNameProd)
      subscription: $(SubscriptionProd)
      deployment: 'prodslotwebdeployment'
      displayName: 'Prod Slot Web Deployment'
      environment: 'OnlineShoppingUIProdSlot'
      GIGYA_CUSTOM_DOMAIN: $(GIGYA_CUSTOM_DOMAIN)
      API_PORT: $(API_PORT)
      API_URL: $(API_URL_PROD)
      APPINSIGHTS_KEY: $(APPINSIGHTS_KEY_PROD)
      APPINSIGHTS_INSTRUMENTATIONKEY: $(APPINSIGHTS_KEY_PROD)
      APPINSIGHTS_SSR_NODENAME: $(APPINSIGHTS_SSR_NODENAME)
      APPINSIGHTS_LIVE: $(APPINSIGHTS_LIVE_PROD)
      DOCKER_CUSTOM_IMAGE_NAME: $(DOCKER_CUSTOM_IMAGE_NAME_PROD)
      DOCKER_SSR_CUSTOM_IMAGE_NAME: $(DOCKER_SSR_CUSTOM_IMAGE_NAME_PROD)
      DOCKER_REGISTRY_SERVER_PASSWORD: $(DOCKER_REGISTRY_SERVER_PASSWORD_PROD)
      DOCKER_REGISTRY_SERVER_URL: $(DOCKER_REGISTRY_SERVER_URL_PROD)
      DOCKER_REGISTRY_SERVER_USERNAME: $(DOCKER_REGISTRY_SERVER_USERNAME_PROD)
      GIGYA_API_KEY: $(GIGYA_API_KEY_PROD)
      GTM_AUTH: $(GTM_AUTH_PROD)
      SITE_LOCATION_API_URL: $(SITE_LOCATION_API_URL_PROD)
      forceContinuation: false
      dockerNamespace: $(ProdDockerRegistry)
      deployToSlotOrASE: true
      slotName: 'preprod'
      ssrSlotName: $(ssrSlotName)
      oliveChatUrl: $(OLIVE_CHAT_WIDGET_URL_PROD)
      citrus_ad_base_url: $(CITRUS_AD_BASE_URL_PROD)

  - template: release/schedule-change-request.yml
    parameters:
      stageName: 'changerequest'
      stageDependency: 'preprodslot'

  - template: release/change-request-in-scheduled.yml
    parameters:
      stageName: 'changerequestscheduled'
      stageDependency: 'changerequest'

  - template: release/change-request-in-progress.yml
    parameters:
      stageName: 'changerequestinprogress'
      stageDependency: 'changerequestscheduled'

  - template: release/appservice-slot-swap.yml
    parameters:
      stageName: 'prodslotswap'
      stageDependency: 'changerequestinprogress'
      resourceGroupName: $(ResourceGroupNameProd)
      subscription: $(SubscriptionProd)
      WebsiteName: $(WebsiteNameProd)
      WebsiteNameSSR: $(WebsiteNameProdSSR)
      targetSlotName: 'production'
      websiteSlotName: 'preprod'
      ssrSlotName: $(ssrSlotName)

  - template: release/close-change-request.yml
    parameters:
      stageName: 'closechangerequest'
      stageDependency: 'prodslotswap'

# Integration AppScan on Cloud (ASoC) and Gitlab
Yaml file and Dockerfile giving ideas in how to integrate ASoC and Gitlab

Scan Job
![image](https://user-images.githubusercontent.com/69405400/144601178-9bc8c675-a2dd-44c4-a312-908800be1472.png)

Artifact downloadable
![image](https://user-images.githubusercontent.com/69405400/144601700-40bfa642-a776-4e4f-ba05-e96f4324ef19.png)

Security Gate response failing or succeeding build
![image](https://user-images.githubusercontent.com/69405400/144601954-ae41e5ea-a9fa-464b-b931-36cd0887723b.png)
![image](https://user-images.githubusercontent.com/69405400/144602140-3e4320f3-a86c-44a1-93ed-5ad7f5fa3348.png)


<b><h1>SAST:</b></h1><br>
Based in 3 components:<br>
1 - a Dockerfile to generate a image container where download ASOC command line client and some tools to be used by Gitlab image Pipeline<br>
2 - YAML project file with a scan job to be used in a YAML project file<br>
3 - some variable that could be on YAML project file or be add directly on Gitlab Project (Settings > CI/CD and expand the Variables)<br>

Dockerfile to generate a docker image with SAClient:</br> 
docker build -t saclient .
````dockerfile
FROM ubuntu:latest
ENV PATH="$HOME/SAClientUtil/bin:${PATH}"
RUN apt update
RUN apt install -y curl unzip maven openjdk-11-jre gradle && apt clean
RUN curl https://cloud.appscan.com/api/SCX/StaticAnalyzer/SAClientUtil?os=linux > $HOME/SAClientUtil.zip
RUN unzip $HOME/SAClientUtil.zip -d $HOME
RUN rm -f $HOME/SAClientUtil.zip
RUN mv $HOME/SAClientUtil.* $HOME/SAClientUtil
````

Gitlab YAML file to run SAST analyzes:
````yaml
image: saclient

# The options to sevSecGw are highIssues, mediumIssues, lowIssues and totalIssues
# maxIssuesAllowed is the amount of issues in selected sevSecGw
# appId is application id located in ASoC 
variables:
  apiKeyId: xxxxxxxxxxxxxxxxxx
  apiKeySecret: xxxxxxxxxxxxxxxxxx
  appId: xxxxxxxxxxxxxxxxxx
  sevSecGw: totalIssues
  maxIssuesAllowed: 200

stages:
- clean
- build
- scan-sast

clean-job:
  stage: clean
  script:
  - gradle clean

build-job:
  stage: build
  script:
  - gradle build

scan-job:
  stage: scan-sast
  script:
  - gradle build
  # Generate IRX files based on source root folder downloaded by Gitlab
  - appscan.sh prepare
  # Authenticate in ASOC
  - appscan.sh api_login -u $apiKeyId -P $apiKeySecret -persist
  # Upload IRX file to ASOC to be analyzed and receive scanId
  - appscan.sh queue_analysis -a $appId >> output.txt
  - cat output.txt
  - scanId=$(sed -n '2p' output.txt)
  # Check Scan Status
  - resultScan=$(appscan.sh status -i $scanId)
  - >
    while true ; do 
      resultScan=$(appscan.sh status -i $scanId)
      echo $resultScan
      if [ "$resultScan" != "Running" ]
        then break
      fi
      sleep 60
    done
  # Get report from ASOC
  - appscan.sh get_result -i $scanId -t html
  # Get summary scan and give it to Security Gateway decision
  - appscan.sh info -i $scanId > scanStatus.txt
  - highIssues=$(cat scanStatus.txt | grep LatestExecution | grep -oP '(?<="NHighIssues":)[^,]*')
  - mediumIssues=$(cat scanStatus.txt | grep LatestExecution | grep -oP '(?<="NMediumIssues":)[^,]*')
  - lowIssues=$(cat scanStatus.txt | grep LatestExecution | grep -oP '(?<="NLowIssues":)[^,]*')
  - totalIssues=$(cat scanStatus.txt | grep LatestExecution | grep -oP '(?<="NIssuesFound":)[^,]*')
  - echo "There is $highIssues high issues, $mediumIssues medium issues and $lowIssues low issues."
  - >
    if [ "$highIssues" -gt "$maxIssuesAllowed" ] && [ "$sevSecGw" == "highIssues" ]
      then
        echo "The company policy permit less than $maxIssuesAllowed $sevSecGw severity"
        echo "Security Gate build failed"
        exit 1
    elif [ "$mediumIssues" -gt "$maxIssuesAllowed" ] && [ "$sevSecGw" == "mediumIssues" ]
      then
        echo "The company policy permit less than $maxIssuesAllowed $sevSecGw severity"
        echo "Security Gate build failed"
        exit 1
    elif [ "$lowIssues" -gt "$maxIssuesAllowed" ] && [ "$sevSecGw" == "lowIssues" ]
      then
        echo "The company policy permit less than $maxIssuesAllowed $sevSecGw severity"
        echo "Security Gate build failed"
        exit 1
    elif [ "$totalIssues" -gt "$maxIssuesAllowed" ] && [ "$sevSecGw" == "totalIssues" ]
      then
        echo "The company policy permit less than $maxIssuesAllowed $sevSecGw severity"
        echo "Security Gate build failed"
        exit 1
    fi
  - echo "The company policy permit less than $maxIssuesAllowed $sevSecGw severity"
  - echo "Security Gate passed"
  
  artifacts:
    when: always
    paths:
      - "*.html"
````

<b><h1>DAST:</b></h1><br>
Based in 2 components:<br>
1 - YAML project file with a scan job to be used in a YAML project file.<br>
2 - some variable that could be on YAML project file or be add directly on Gitlab Project (Settings > CI/CD and expand the Variables)<br>

Gitlab YAML file to run DAST analyzes:
````yaml
# The options to sevSecGw are highIssues, mediumIssues, lowIssues and totalIssues.
# maxIssuesAllowed is the amount of issues in selected sevSecGw.
# appId is application id located in ASoC.
# appscanPresenceId is AppScan Presence ID that will be used to reach out URL.
# About scan authenticated, just add the login recorded file "dast.config" in root of source code that it will be sent to ASOC. 
variables:
  apiKeyId: xxxxxxxxxxxxxxxxxx
  apiKeySecret: xxxxxxxxxxxxxxxxxx
  appId: xxxxxxxxxxxxxxxxxx
  appscanPresenceId: xxxxxxxxxxxxxxxxxx
  urlTarget: http://demo.testfire.net/
  sevSecGw: totalIssues
  maxIssuesAllowed: 200

stages:
- scan-dast

scan-job:
  stage: scan-dast
  script:
  # 1 - Authenticate to ASOC and get Token. 
  # 2 - If available on root source code, get dast.config file, upload do ASOC and get dastFileId. 
  # 3 - Request a DAST Scan to ASOC. There is some report config that could be passed on json config or it could be a variable. 
  - >
    asocToken=$(curl -s -X POST --header "Content-Type: application/json" --header "Accept: application/json" -d '{"KeyId":"'"${apiKeyId}"'","KeySecret":"'"${apiKeySecret}"'"}' 'https://cloud.appscan.com/api/V2/Account/ApiKeyLogin' | grep -oP '(?<="Token":")[^"]*')
    dastFileId=$(curl -X 'POST' 'https://cloud.appscan.com/api/v2/FileUpload' -H 'accept: application/json' -H "Authorization: Bearer $asocToken" -H 'Content-Type: multipart/form-data' -F 'fileToUpload=@dast.config;type=application/xml' | grep -oP '(?<="FileId":")[^"]*')
    data=$(date '+%m-%d-%Y')
    scanId=$(curl -s -X 'POST' 'https://cloud.appscan.com/api/v2/Scans/DynamicAnalyzerWithFiles' -H 'accept: application/json' -H "Authorization: Bearer $asocToken" -H 'Content-Type: application/json' -d  '{"StartingUrl":"'"$urlTarget"'","TestOnly":false,"ExploreItems":[],"LoginUser":"","LoginPassword":"","TestPolicy":"Default.policy","ExtraField":"","ScanType":"Staging","PresenceId":"'"$appscanPresenceId"'","IncludeVerifiedDomains":false,"HttpAuthUserName":"","HttpAuthPassword":"","HttpAuthDomain":"","TestOptimizationLevel":"Fastest","LoginSequenceFileId":"'"$dastFileId"'","ThreadNum":10,"ConnectionTimeout":null,"UseAutomaticTimeout":true,"MaxRequestsIn":null,"MaxRequestsTimeFrame":null,"ScanName":"'"DAST $date $urlTarget"'","EnableMailNotification":false,"Locale":"en","AppId":"'"$appId"'","Execute":true,"Personal":false,"ClientType":"user-site","Comment":null,"FullyAutomatic":false,"RecurrenceRule":null,"RecurrenceStartDate":null}' | jq -r '. | {Id} | join(" ")')
# Check DAST Scan Status. When Ready exit loop.
  - >  
    for x in $(seq 1 1000)
      do
        scanStatus=$(curl -s -X 'GET' "https://cloud.appscan.com/api/v2/Scans/$scanId" -H 'accept: application/json' -H "Authorization: Bearer $asocToken" | jq -r '.LatestExecution | {Status} | join(" ")')
        echo $scanStatus 
        if [ "$scanStatus" == "Ready" ]
          then break
        elif [ "$scanStatus" == "Failed" ] 
          then
            echo "Scan Failed. Check ASOC logs"
            exit 1
        fi
        sleep 60
      done
# Request for Report in HTML. There is some report config that could be passed on json config.   
  - >  
    reportId=$(curl -s -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' --header "Authorization: Bearer $asocToken" -d '{"Configuration":{"Summary":true,"Details":true,"Discussion":true,"Overview":true,"TableOfContent":true,"Articles":true,"History":true,"Coverage":true,"MinimizeDetails":true,"ReportFileType":"HTML","Title":"","Notes":"","Locale":"en"},"OdataFilter":"","ApplyPolicies":"None"}' "https://cloud.appscan.com/api/v2/Reports/Security/Scan/$scanId" | grep -oP '(?<="Id":")[^"]*')
# Loop to get report file.
  - >
    for x in {1..30}
      do
        curl -s -X GET --header 'Accept: text/xml' --header "Authorization: Bearer $asocToken" "https://cloud.appscan.com/api/v2/Reports/Download/$reportId" > DAST_report.html
        if [[ -s DAST_report.html ]] 
      then
          break
      fi
      sleep 1
    done
# Get summary scan and give it to Security Gateway decision    
  - >
    curl -X 'GET' "https://cloud.appscan.com/api/v2/Scans/$scanId" -H 'accept: application/json' -H "Authorization: Bearer $asocToken" > scanResult.txt
  - highIssues=$(cat scanResult.txt | jq -r '.LastSuccessfulExecution | {NHighIssues} | join(" ")')
  - mediumIssues=$(cat scanResult.txt | jq -r '.LastSuccessfulExecution | {NMediumIssues} | join(" ")')
  - lowIssues=$(cat scanResult.txt | jq -r '.LastSuccessfulExecution | {NLowIssues} | join(" ")')
  - totalIssues=$(cat scanResult.txt | jq -r '.LastSuccessfulExecution | {NIssuesFound} | join(" ")')
  - >
    if [ "$highIssues" -gt "$maxIssuesAllowed" ] && [ "$sevSecGw" == "highIssues" ]
      then
        echo "There is $highIssues high issues, $mediumIssues medium issues and $lowIssues low issues"
        echo "The company policy permit less than $maxIssuesAllowed $sevSecGw severity"
        echo "Security Gate build failed"
        exit 1
    elif [ "$mediumIssues" -gt "$maxIssuesAllowed" ] && [ "$sevSecGw" == "mediumIssues" ]
      then
        echo "There is $highIssues high issues, $mediumIssues medium issues and $lowIssues low issues"
        echo "The company policy permit less than $maxIssuesAllowed $sevSecGw severity"
        echo "Security Gate build failed"
        exit 1
    elif [ "$lowIssues" -gt "$maxIssuesAllowed" ] && [ "$sevSecGw" == "lowIssues" ]
      then
        echo "There is $highIssues high issues, $mediumIssues medium issues and $lowIssues low issues"
        echo "The company policy permit less than $maxIssuesAllowed $sevSecGw severity"
        echo "Security Gate build failed"
        exit 1
    elif [ "$totalIssues" -gt "$maxIssuesAllowed" ] && [ "$sevSecGw" == "totalIssues" ]
      then
        echo "There is $highIssues high issues, $mediumIssues medium issues and $lowIssues low issues"
        echo "The company policy permit less than $maxIssuesAllowed $sevSecGw severity"
        echo "Security Gate build failed"
        exit 1
    fi
  - echo "There is $highIssues high issues, $mediumIssues medium issues and $lowIssues low issues"
  - echo "The company policy permit less than $maxIssuesAllowed $sevSecGw severity"
  - echo "Security Gate passed"

  artifacts:
    when: always
    paths:
      - "*.html"
````

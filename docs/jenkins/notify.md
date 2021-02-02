## **钉钉脚本**

```
#!/usr/local/python-3.6/bin/python3.6
#钉钉发版通知脚本

import json,requests,sys,time,datetime

#[工程名、分支号、是否成功、编号]
job = sys.argv[1]
branch = sys.argv[2]
stat = sys.argv[3]
bianhao = sys.argv[4]
user_name = sys.argv[5]

if int(stat) == 0:
    name = '发布成功！'
    tupian = '![file](https://gd-prod-private.oss-cn-beijing.aliyuncs.com/yw/QQ%E6%88%AA%E5%9B%BE20201109153327.png)'
elif int(stat) == 1:
    name = '发布失败！'
    tupian = '![file](https://gd-prod-private.oss-cn-beijing.aliyuncs.com/yw/QQ%E6%88%AA%E5%9B%BE20201109163201.png)'

url = "https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxxxxxxxxxxxxxxxxxx"
title = job + name
nowtime = datetime.datetime.now()
nowtime = str(nowtime.strftime('%Y-%m-%d %H:%M:%S'))
msg = """### %s \n
> 时间: %s \n
> 分支: %s \n
> 提交人：%s \n
> 编号: #%s \n
> 地址: [工程链接](http://1.1.1.1:8080/job/%s) \n
%s
"""

def Alert():
    headers = {"Content-Type": "application/json"}
    data = {"msgtype": "markdown", 
        "markdown": {
            "title": title,
            "text": msg %(title, nowtime, branch, user_name, bianhao, job, tupian)
        }
    }

    r = requests.post(url, data=json.dumps(data), headers=headers, verify=False)
    print(r.text)
Alert()

```

```
post {
        success {
            wrap ([$class: 'BuildUser']) {
                 script {
                def user_user = env.BUILD_USER_ID
                currentBuild.description = "executor is #${user_user}#" 
            }
                sh "/data/jenkins_home/jenkins_work/jenkins_script/dingding.py ${JOB_NAME} ${BranchName} 0 ${BUILD_NUMBER} ${BUILD_USER}"
        }
    }
        failure {
            wrap ([$class: 'BuildUser']) {
                sh "/data/jenkins_home/jenkins_work/jenkins_script/dingding.py ${JOB_NAME} ${BranchName} 1 ${BUILD_NUMBER} ${BUILD_USER}"
        }
     }
    }
```
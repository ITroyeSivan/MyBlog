# 介绍

一个关于AWS云安全的渗透靶场。

# 1.Breach in the Cloud

场景：

我们已接到潜在安全事件的警报。庞大物流安全团队为您提供了一个看到异常活动的账户的AWS密钥，以及活动期间的AWS CloudTrail日志。我们需要您的专业知识，通过分析我们的CloudTrail日志，识别受损的AWS服务和任何被过滤的数据来确认漏洞。

学习成果：

- 对JSON文件进行预处理以便于分析
- 熟悉AWS CLI
- 熟悉分析CloudTrail日志
- 枚举S3存储桶
- 模拟攻击者验证入侵路径

真实世界背景：

分析AWS CloudTrail日志是检测AWS帐户内可疑活动的标准做法，而S3存储桶由于可能包含有价值的数据而经常成为攻击者的目标。

流程如下：

首先下载INCIDENT-3252.zip

在Nano或Vim等文本编辑器中打开文件会发现JSON文件没有经过修饰，这使它们更难阅读。为了美化文件，我们可以使用jq——一个用于解析和结构化数据的命令行JSON处理器。如果需要，使用命令apt-Install-jq进行安装。然后在包含JSON文件的当前目录中发出以下命令。

我这里用python将其转为可阅读的格式：

```python
import json


def pretty_print_json_to_file(input_file_path, output_file_path):
    try:
        # 读取JSON日志文件
        with open(input_file_path, 'r') as input_file:
            log_data = json.load(input_file)

            # 格式化JSON数据
            formatted_data = json.dumps(log_data, indent=4)

            # 将格式化后的数据写入输出文件
            with open(output_file_path, 'w') as output_file:
                output_file.write(formatted_data)

            print(f"格式化后的日志已写入文件：{output_file_path}")

    except FileNotFoundError:
        print(f"文件 {input_file_path} 未找到。")
    except json.JSONDecodeError:
        print(f"文件 {input_file_path} 不是有效的JSON格式。")


# 输入文件路径和输出文件路径
input_file_path = './tmp/107513503799_CloudTrail_us-east-1_20230826T2035Z_PjmwM7E4hZ6897Aq.json'
output_file_path = './output/20230826T2035Z.json'

# 调用函数
pretty_print_json_to_file(input_file_path, output_file_path)
```


# 场景2: 基于 ModelEngine 下游任务训练

## 准备工作

### 准备医院数据

首先，需要获得医院病理系统 PIS 系统（病理科病例管理系统）和 MIS 系统（病理科影像管理系统）的连接和查询方式。DataMate 支持关系型数据库的表/视图查询，亦可以选择使用JSON数据格式的 RESTful API 接口进行查询。两种方法所需的连接和查询信息如下：

1. 数据库查询：
   1. 数据库连接URL、用户名（User）、密码（Password）
   2. SQL查询语句（Query）
2. API查询：
   1. 接口地址（URL）、请求方式（GET/POST/）
   2. 请求体参数（Body）、请求头参数（Header）
   3. 数据解析（Schema）

我们需要从 PIS 系统和 MIS 系统中获取的数据如下：
1. 从PIS系统获取的数据：以病例为维度，病理号、病理报告、取材部位
2. 从MIS系统获取的数据：以数字病理切片为维度，病理号、病理切片号、数字切片路径、缩略图路径（当数字切片为SDPC格式时需要）

### 准备 DataMate 工作环境

病理系统数据预处理算子准备：从 [DataMate-Ops](https://github.com/JasonW404-HW/DataMate-Ops) 下载 main 分支代码为 zip 文件，解压后将 patho_sys_preprocess/ 文件夹下所有内容，打包为 .zip 或 .tar 压缩包文件，确保压缩包名称为 patho_sys_preprocess.zip 。进入 DataMate - 算子市场 页面，将该算子上传到 DataMate 算子市场 中，并发布。

DataMate 运行环境变更：因为数字病理切片的文件体积较大，且数量较多，本最佳实践中采用的数据准备方式为文件软连接形式。即，数字病理切片及其缩略图并不会传输到 DataMate 的服务器或关联的存储中，而是通过将 医院病理系统图像存储 挂载到 DataMate 运行环境中，通过创建文件软链接进行关联管理的方法。因此，需要做如下环境变更操作：

1. 在主机上完成对 医院病理系统图像存储 的挂载，建议挂载在 `/mnt` 目录下。注意，如果是多主机集群，请确保所有主机上都完成了挂载。
2. 修改 DataMate 安装部署配置，将 医院病理系统图像存储 在主机上的路径，挂载到 POD 中，并记录该存储在 POD 中的路径。需要挂载的 POD 包括：`datamate-backend`, `datamate-runtime`, `datamate-backend-python`, `label-studio`。

具体的操作步骤如下：

医院病理图像存储在主机 `/mnt` 下的某个挂载点（例如 `/mnt/pathology`），需要以 `hostPath` 方式挂载到4个 POD 中。按照当前 Helm 配置架构，所有修改集中在如下文件中：

- `deployment/helm/datamate/values.yaml`：控制backend、runtime、backend-python
- `deployment/helm/label-studio/values.yaml`：控制 label-studio
- `deployment/helm/label-studio/templates/deployment.yaml`：label-studio 的 deployment 模板（需要添加 hostPath volume 支持）

1. 确认主机挂载点路径：在每个 Kubernetes 节点上确认病理图像的实际路径。假设实际路径为 `/mnt/pathology`，POD 内挂载路径统一使用 `/data/pathology`。
   
    ```bash
    ls /mnt/pathology   # 或 /mnt/wsi-images 等实际挂载点名称
    ```

2. 修改 datamate 主 values.yaml：
在 values.yaml 中，在顶部 YAML anchor 区域新增一个 hostPath volume anchor，然后分别追加到 backend、backend-python、runtime 的 volumes 和 volumeMounts 中：

    ```yaml
    # 在现有 operatorVolume anchor 后新增：
    pathologyVolume: &pathologyVolume
    name: pathology-volume
    hostPath:
        path: /mnt/pathology   # ← 替换为实际主机路径
        type: Directory

    # 在 backend.volumes 中追加：
    backend:
    volumes:
        - *datasetVolume
        - *flowVolume
        - *logVolume
        - *operatorVolume
        - *pathologyVolume        # ← 新增
    volumeMounts:
        - name: dataset-volume
        mountPath: /dataset
        - name: flow-volume
        mountPath: /flow
        - name: log-volume
        mountPath: /var/log/datamate
        - name: operator-volume
        mountPath: /operators
        - name: pathology-volume  # ← 新增
        mountPath: /data/pathology
        readOnly: true          # 建议只读，避免误操作
    # backend-python 和 runtime 同理追加相同两项
    ```
3. 第三步：修改 label-studio 的 values.yaml 和 deployment 模板
    values.yaml 新增：
    ```yaml
    pathologyVolume:
    enabled: true
    hostPath: /mnt/pathology   # ← 替换为实际主机路径
    mountPath: /label-studio/local/pathology
    ```

    deployment.yaml 中追加：

    ```yaml
    # volumeMounts 区域追加：
            - name: pathology
                mountPath: /label-studio/local/pathology
                readOnly: true   # 建议只读

    # volumes 区域追加：
            {{- if .Values.pathologyVolume.enabled }}
            - name: pathology
            hostPath:
                path: {{ .Values.pathologyVolume.hostPath }}
                type: Directory
            {{- end }}
    ```

    注意：label-studio 的 `LOCAL_FILES_DOCUMENT_ROOT` 已设为 `/label-studio/local`，病理图像放在其子目录 `/label-studio/local/pathology` 下，label-studio 可以直接通过 Local Storage 访问。

4. 确认各 POD 中的存储路径

    |POD                     | 主机路径       | POD 内路径                    | 权限 |
    |------------------------|----------------|-------------------------------|------|
    |datamate-backend        | /mnt/pathology | /data/pathology               | 只读 |
    |datamate-runtime        | /mnt/pathology | /data/pathology               | 只读 |
    |datamate-backend-python | /mnt/pathology | /data/pathology               | 只读 |
    |label-studio            | /mnt/pathology | /label-studio/local/pathology | 只读 |

## 训练数据准备

### 数据归集

这一部分的操作目的，是为了将医院病理科业务系统中的数据，以结构化数据的形式，导入到 DataMate 中，便于后续处理和使用。

1. 进入 DataMate - 数据归集 页面，点击 创建任务 。
2. 填入任务名称（不影响任务执行），根据数据查询的方式，选择正确的归集模板：
   1. 通过API查询：选择 API 归集模板
   2. 通过关系型数据库查询：选择 MySQL 归集模板 或 通用关系型数据库归集模板 。
3. 等待归集结束。进入 DataMate - 数据管理 页面，点击 创建数据集。
4. 填写数据集名称，选择合适的数据集类型（文本），关联 步骤 1-3 中创建并运行完成的数据归集任务。点击 确定，创建数据集。

> ‼️ **注意**：为了后续归集数据的清洗能正确执行，请对PIS系统和MIS系统分别设置归集任务，并将归集结果写入同一数据集中。

**预期结果**: 在对应数据集中，能看到两个 CSV 文件。

### 数据清洗

1. 进入 DataMate - 数据处理 页面，点击 创建模板 ，创建 “病理数据归集清洗” 模板。注意，该模板中仅需勾选一个“病理系统数据预处理”算子。保存并创建模板。
2. 进入 DataMate - 数据处理 页面，点击 创建任务 ，创建 “病理图片归集“任务。
3. 选择之前创建的“病理数据归集清洗”模板，创建清洗任务，等待清洗完成。

**预期结果**：在目标数据集“WSI归集”中，可以看到有若干 数字病理切片 和 数字病理切片的预览图 （如果有SDPC文件）。

> 至此，病理场景数据准备已经完成，您可以进入下一步 *模型训练* 。

## 模型训练

# lobechat-deploy-action

## 问题
- 当前构建的镜像无法上传
- 我需要同时构建arm和amd64的docker镜像
## 补充
- 尽可能在原基础上做修改
- 官方的workflow构建流程很棒，可以参照

## log
``` 
#28 [app 11/11] RUN     addgroup -S -g 1001 nodejs     && adduser -D -G nodejs -H -S -h /app -u 1001 nextjs     && chown -R nextjs:nodejs /app /etc/proxychains4.conf
#28 DONE 12.2s

#29 [stage-3 1/1] COPY --from=app / /
#29 DONE 2.5s

#30 exporting to image
#30 exporting layers
#30 exporting layers 17.2s done
#30 exporting manifest sha256:3b63679e1c4c43e4fd7ee6383b9c241fec22979db809a49167e1ce3cd108a5da done
#30 exporting config sha256:74667d094e6c008cd76d0d46487b61249e8d752261f61fb269a641c900359523 done
#30 exporting attestation manifest sha256:8f5708b82dbd3c3d4f1fa402ffaa3a4b44c22ed58e88af704be613a983ecd363 done
#30 exporting manifest list sha256:6b06dadf7d1d541406d636a85709aca3758a389bdf89b366b3aa92f44a477aad done
#30 ERROR: failed to push ***/lobe-chat-clerk:1.82.1: can't push tagged ref docker.io/***/lobe-chat-clerk:1.82.1 by digest
------
 > exporting to image:
------
ERROR: failed to solve: failed to push ***/lobe-chat-clerk:1.82.1: can't push tagged ref docker.io/***/lobe-chat-clerk:1.82.1 by digest
Error: buildx failed with: ERROR: failed to solve: failed to push ***/lobe-chat-clerk:1.82.1: can't push tagged ref docker.io/***/lobe-chat-clerk:1.82.1 by digest
```

## workflow
``` yml
name: Build_Docker_Image

on:
  workflow_dispatch: # 允许手动触发构建特定版本
    inputs:
      release_tag:
        description: 'Upstream lobehub/lobe-chat release tag to build'
        required: true
        default: 'latest' # 注意：'latest' 可能需要特殊处理，最好是具体版本号
  workflow_call: # 由 monitor-upstream.yml 触发
    inputs:
      release_tag:
        description: 'Upstream release tag from monitor workflow'
        required: true
        type: string
    # 如果需要 secrets，可以在这里定义并传递
    # secrets:
    #   DOCKER_REGISTRY_USER:
    #     required: true
    #   DOCKER_REGISTRY_PASSWORD:
    #     required: true
    #   NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY:
    #     required: false # 根据你的需求设置是否必须
    #   CLERK_SECRET_KEY:
    #     required: false
    #   CLERK_WEBHOOK_SECRET:
    #     required: false

env:
  # 定义环境变量，如果不用 workflow_call 的 secrets 传递，则定义在这里
  REGISTRY_IMAGE: ${{ secrets.DOCKER_REGISTRY_USER }}/lobe-chat-clerk # 确保 secrets.DOCKER_REGISTRY_USER 可用
  UPSTREAM_REPO_OWNER: "lobehub"
  UPSTREAM_REPO_NAME: "lobe-chat"
  # BUILD_CONTEXT_DIR: "build-context" # 构建上下文目录名

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}-${{ github.event.inputs.release_tag || github.sha }}
  cancel-in-progress: true

jobs:
  build-and-publish:
    strategy:
      fail-fast: true # 一个平台失败不取消其他平台
      matrix:
        include:
          - platform: linux/amd64
            os: ubuntu-latest
          - platform: linux/arm64
            # 注意: GitHub 托管的 ARM runner 可能需要特定标签，或使用自托管 runner
            # os: ubuntu-24.04-arm # 确认 Runner 标签可用性
            os: ubuntu-latest # 如果 ARM runner 不稳定或不可用，可暂时都在 ubuntu-latest 上用 QEMU 构建
    runs-on: ${{ matrix.os }}
    name: Build ${{ matrix.platform }} Image
    permissions:
      contents: read # 读取仓库中的 custom/ 目录
      packages: write # 推送 Docker 镜像

    steps:
      - name: Harden Runner
        uses: step-security/harden-runner@v2 # v2.9.0
        with:
          egress-policy: audit # TODO: change to 'egress-policy: block' after couple of runs

      - name: Checkout Workflow Repository (for custom files)
        uses: actions/checkout@v4
        with:
          path: workflow-repo # 将本仓库代码 checkout 到独立目录

      - name: Validate Release Tag Input
        id: vars
        run: |
          TAG="${{ inputs.release_tag }}"
          # 简单验证，确保 tag 不为空
          if [ -z "$TAG" ]; then
            echo "Error: Release tag input is empty."
            exit 1
          fi
          # 可以添加更复杂的验证，例如检查格式
          echo "Building for upstream tag: $TAG"
          # 获取当前 Git SHA 用于可能的调试标签
          echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
          echo "release_tag=$TAG" >> $GITHUB_OUTPUT

      - name: Set up QEMU (needed if building ARM on AMD64 host)
        if: runner.os == 'Linux' && matrix.platform != 'linux/amd64' # 仅在需要跨平台模拟时设置
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        # with:
        #   driver-opts: image=moby/buildkit:v0.13.0 # 可选：指定 buildkit 版本

      - name: Prepare Build Context
        id: prepare_context
        run: |
          # 替换斜杠为连字符以避免路径问题
          PLATFORM_NAME="${{ matrix.platform }}"
          SAFE_PLATFORM_NAME="${PLATFORM_NAME//\//-}"  # 将 linux/amd64 替换为 linux-amd64
          BUILD_CONTEXT_DIR="build-context-${SAFE_PLATFORM_NAME}"
          RELEASE_TAG="${{ steps.vars.outputs.release_tag }}"
          SOURCE_ARCHIVE="lobe-chat-source-$RELEASE_TAG.tar.gz"
          
          # 获取工作目录的绝对路径
          WORKSPACE_DIR="${GITHUB_WORKSPACE}"
          CUSTOM_DIR="${WORKSPACE_DIR}/workflow-repo/custom"
          
          echo "Creating build context directory: $BUILD_CONTEXT_DIR"
          mkdir -p "$BUILD_CONTEXT_DIR"
          cd "$BUILD_CONTEXT_DIR"
          
          echo "Downloading upstream source for tag v$RELEASE_TAG..."
          curl -sL "https://github.com/${{ env.UPSTREAM_REPO_OWNER }}/${{ env.UPSTREAM_REPO_NAME }}/archive/refs/tags/v$RELEASE_TAG.tar.gz" -o "$SOURCE_ARCHIVE"
          
          echo "Extracting source code..."
          tar -xzf "$SOURCE_ARCHIVE" --strip-components=1
          rm "$SOURCE_ARCHIVE"
          
          echo "Applying custom modifications..."
          echo "Custom directory: ${CUSTOM_DIR}"
          # 使用绝对路径指向 custom 目录
          if [ -d "${CUSTOM_DIR}" ]; then
            rsync -av "${CUSTOM_DIR}/" ./ --exclude '.gitkeep'
            echo "Custom files copied successfully"
          else
            echo "Warning: Custom directory not found at ${CUSTOM_DIR}"
            # 列出工作目录结构以帮助调试
            echo "Workspace structure:"
            ls -la "${WORKSPACE_DIR}"
            echo "Workflow repo structure:"
            ls -la "${WORKSPACE_DIR}/workflow-repo"
          fi
          
          echo "Build context prepared in $PWD"
          cd "${WORKSPACE_DIR}" # 返回工作目录
          echo "context_path=$BUILD_CONTEXT_DIR" >> $GITHUB_OUTPUT

      - name: Docker meta tags
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY_IMAGE }}
          tags: |
            # 主要标签：上游版本号
            type=raw,value=${{ steps.vars.outputs.release_tag }}
            # 可选：如果这是最新的上游版本，也标记为 latest (需要 monitor workflow 传递是否最新的信息，或者在这里再次查询)
            # type=raw,value=latest,enable={{trigger_type == 'schedule' and is_latest == 'true'}} # 这需要更复杂的逻辑
            # 为手动触发添加标签（如果需要区分）
            type=raw,value=${{ steps.vars.outputs.release_tag }}-manual,enable=${{ github.event_name == 'workflow_dispatch' }}
            # 可以添加基于 Git SHA 的调试标签
            type=sha,prefix=${{ steps.vars.outputs.release_tag }}-,suffix=-${{ matrix.platform }},format=short,enable=true

      - name: Login to Docker Registry
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_REGISTRY_USER }} # 确保 secrets 可用
          password: ${{ secrets.DOCKER_REGISTRY_PASSWORD }}
          registry: docker.io
          
          # registry: ghcr.io # 如果使用 GHCR

      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: ${{ steps.prepare_context.outputs.context_path }}
          # 指定被替换后的 Dockerfile 路径
          file: ${{ steps.prepare_context.outputs.context_path }}/Dockerfile.database
          platforms: ${{ matrix.platform }}
          push: true # 直接构建并推送单平台镜像
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            SHA=${{ steps.vars.outputs.sha_short }}
            UPSTREAM_VERSION=${{ steps.vars.outputs.release_tag }}
            # 传递你的 secrets 作为 build args
            NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=${{ secrets.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY }}
            CLERK_SECRET_KEY=${{ secrets.CLERK_SECRET_KEY }}
            CLERK_WEBHOOK_SECRET=${{ secrets.CLERK_WEBHOOK_SECRET }}
          # 输出类型改为 image，因为我们将合并 manifest
          outputs: type=image,name=${{ env.REGISTRY_IMAGE }},push-by-digest=true,name-canonical=true


```

## lobe-chat官方workflow
``` yml
name: Publish Database Docker Image

on:
  workflow_dispatch:
  release:
    types: [published]
  pull_request:
    types: [synchronize, labeled, unlabeled]

concurrency:
  group: ${{ github.ref }}-${{ github.workflow }}
  cancel-in-progress: true

env:
  REGISTRY_IMAGE: lobehub/lobe-chat-database
  PR_TAG_PREFIX: pr-

jobs:
  build:
    # 添加 PR label 触发条件
    if: |
      (github.event_name == 'pull_request' &&
       contains(github.event.pull_request.labels.*.name, 'Build Docker')) ||
      github.event_name != 'pull_request'

    strategy:
      matrix:
        include:
          - platform: linux/amd64
            os: ubuntu-latest
          - platform: linux/arm64
            os: ubuntu-24.04-arm
    runs-on: ${{ matrix.os }}
    name: Build ${{ matrix.platform }} Image
    steps:
      - name: Prepare
        run: |
          platform=${{ matrix.platform }}
          echo "PLATFORM_PAIR=${platform//\//-}" >> $GITHUB_ENV

      - name: Checkout base
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # 为 PR 生成特殊的 tag
      - name: Generate PR metadata
        if: github.event_name == 'pull_request'
        id: pr_meta
        run: |
          branch_name="${{ github.head_ref }}"
          sanitized_branch=$(echo "${branch_name}" | sed -E 's/[^a-zA-Z0-9_.-]+/-/g')
          echo "pr_tag=${sanitized_branch}-$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY_IMAGE }}
          tags: |
            # PR 构建使用特殊的 tag
            type=raw,value=${{ env.PR_TAG_PREFIX }}${{ steps.pr_meta.outputs.pr_tag }},enable=${{ github.event_name == 'pull_request' }}
            # release 构建使用版本号
            type=semver,pattern={{version}},enable=${{ github.event_name != 'pull_request' }}
            type=raw,value=latest,enable=${{ github.event_name != 'pull_request' }}

      - name: Docker login
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_REGISTRY_USER }}
          password: ${{ secrets.DOCKER_REGISTRY_PASSWORD }}

      - name: Get commit SHA
        if: github.ref == 'refs/heads/main'
        id: vars
        run: echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Build and export
        id: build
        uses: docker/build-push-action@v5
        with:
          platforms: ${{ matrix.platform }}
          context: .
          file: ./Dockerfile.database
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            SHA=${{ steps.vars.outputs.sha_short }}
          outputs: type=image,name=${{ env.REGISTRY_IMAGE }},push-by-digest=true,name-canonical=true,push=true

      - name: Export digest
        run: |
          rm -rf /tmp/digests
          mkdir -p /tmp/digests
          digest="${{ steps.build.outputs.digest }}"
          touch "/tmp/digests/${digest#sha256:}"

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: digest-${{ env.PLATFORM_PAIR }}
          path: /tmp/digests/*
          if-no-files-found: error
          retention-days: 1

  merge:
    name: Merge
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout base
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Download digests
        uses: actions/download-artifact@v4
        with:
          path: /tmp/digests
          pattern: digest-*
          merge-multiple: true

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # 为 merge job 添加 PR metadata 生成
      - name: Generate PR metadata
        if: github.event_name == 'pull_request'
        id: pr_meta
        run: |
          branch_name="${{ github.head_ref }}"
          sanitized_branch=$(echo "${branch_name}" | sed -E 's/[^a-zA-Z0-9_.-]+/-/g')
          echo "pr_tag=${sanitized_branch}-$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY_IMAGE }}
          tags: |
            type=raw,value=${{ env.PR_TAG_PREFIX }}${{ steps.pr_meta.outputs.pr_tag }},enable=${{ github.event_name == 'pull_request' }}
            type=semver,pattern={{version}},enable=${{ github.event_name != 'pull_request' }}
            type=raw,value=latest,enable=${{ github.event_name != 'pull_request' }}

      - name: Docker login
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_REGISTRY_USER }}
          password: ${{ secrets.DOCKER_REGISTRY_PASSWORD }}

      - name: Create manifest list and push
        working-directory: /tmp/digests
        run: |
          docker buildx imagetools create $(jq -cr '.tags | map("-t " + .) | join(" ")' <<< "$DOCKER_METADATA_OUTPUT_JSON") \
            $(printf '${{ env.REGISTRY_IMAGE }}@sha256:%s ' *)

      - name: Inspect image
        run: |
          docker buildx imagetools inspect ${{ env.REGISTRY_IMAGE }}:${{ steps.meta.outputs.version }}
```
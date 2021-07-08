1. build-base 制作编译docker(centos)
    A. `docker build --build-arg IDOEWODOCKERREPO=${IDOEWODOCKERREPO} --build-arg HTTP_PROXY=${HTTP_PROXY} \
	--build-arg HTTPS_PROXY=${HTTPS_PROXY} --build-arg CNFVPP_WORK_ENV_OS_NAME=${CNFVPP_WORK_ENV_OS_NAME} \
	--build-arg CNFVPP_WORK_ENV_OS_VERSION=${CNFVPP_WORK_ENV_OS_VERSION} --rm -t ${CNFVPP_BUILD_BASE_IMAGE_NAME} \
	-f ${CNFVPPROOT}/build/Dockerfile.${CNFVPP_WORK_ENV_OS_NAME}${CNFVPP_WORK_ENV_OS_VERSION}.cnf-vpp.build-base ./`
    _使用docker file创建docker image,并同时生成EVL参数供程序运行使用_

    B. push_container.sh ${IDOEWODOCKERREPO} ${CNFVPP_BUILD_BASE_IMAGE_NAME} ${CNFVPP_BUILD_BASE_IMAGE_VERSION}

    _上传docker image到docker hub, 其中$IDOEWODOCKERREPO由用户预定义，且docke login已成功登录_
2. pkg 编译应用程序包（依赖于build-base)
    A. docker run --name ${CNFVPP_BUILD_BASE_IMAGE_NAME}-${CNFVPP_BUILD_BASE_IMAGE_VERSION} --user root:root --env GO111MODULE \
	--env XDG_CACHE_HOME=/tmp/.cache -v ${IDOEWOROOT}:/opt/ido-ewo/ -dit \
	${IDOEWODOCKERREPO}${CNFVPP_BUILD_BASE_IMAGE_NAME}:${CNFVPP_BUILD_BASE_IMAGE_VERSION} /bin/sh

    _将第1步制作的docker(centos)启动，并将当前Host源码目录挂载到Guest /opt/ido-ewo目录下_

    B. ocker exec ${CNFVPP_BUILD_BASE_IMAGE_NAME}-${CNFVPP_BUILD_BASE_IMAGE_VERSION} /bin/sh -c \
	"cd /opt/ido-ewo/platform/cnf-vpp;make -C src install-dep;make -C src pkg"

    _在guest中调用make命令，即VPP相关依赖安装和编译，生成pkg包,此步骤同时包含VPP集成组件的编译，如sweetcomb/RestAPI等_

    C. cp ${CNFVPPROOT}/src/sweetcomb/${PKG}/*.${PKG} ${CNFVPPROOT}/src/vpp/vpp-20.09/build-root/*vpp*.${PKG} \
	${CNFVPPROOT}/src/sweetcomb/build-root/build/sweetcomb*.${PKG} ${CNFVPPROOT}/build/packages

    _将编译结果拷贝至Host build/packages目录_

3. service-base 制作发布的docker应用（依赖于编译出的deb/rpm包）


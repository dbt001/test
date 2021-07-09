# Makefile分析
## 前半部分-生成可运行程序
### build-base 制作编译docker(centos)
1. docker build --build-arg IDOEWODOCKERREPO=${IDOEWODOCKERREPO} --build-arg HTTP_PROXY=${HTTP_PROXY} \
	--build-arg HTTPS_PROXY=${HTTPS_PROXY} --build-arg CNFVPP_WORK_ENV_OS_NAME=${CNFVPP_WORK_ENV_OS_NAME} \
	--build-arg CNFVPP_WORK_ENV_OS_VERSION=${CNFVPP_WORK_ENV_OS_VERSION} --rm -t ${CNFVPP_BUILD_BASE_IMAGE_NAME} \
	-f ${CNFVPPROOT}/build/Dockerfile.${CNFVPP_WORK_ENV_OS_NAME}${CNFVPP_WORK_ENV_OS_VERSION}.cnf-vpp.build-base ./  
    _使用docker file创建docker image,并同时生成EVL参数供程序运行使用_  
2. push_container.sh ${IDOEWODOCKERREPO} ${CNFVPP_BUILD_BASE_IMAGE_NAME} ${CNFVPP_BUILD_BASE_IMAGE_VERSION}  
    _上传docker image到docker hub, 其中$IDOEWODOCKERREPO由用户预定义，且docke login已成功登录_  
### pkg 编译应用程序包（依赖于build-base)
1. docker run --name ${CNFVPP_BUILD_BASE_IMAGE_NAME}-${CNFVPP_BUILD_BASE_IMAGE_VERSION} --user root:root --env GO111MODULE \
	--env XDG_CACHE_HOME=/tmp/.cache -v ${IDOEWOROOT}:/opt/ido-ewo/ -dit \
	${IDOEWODOCKERREPO}${CNFVPP_BUILD_BASE_IMAGE_NAME}:${CNFVPP_BUILD_BASE_IMAGE_VERSION} /bin/sh  
    _将第1步制作的docker(centos)启动，并将当前Host源码目录挂载到Guest /opt/ido-ewo目录下_  
2. docker exec ${CNFVPP_BUILD_BASE_IMAGE_NAME}-${CNFVPP_BUILD_BASE_IMAGE_VERSION} /bin/sh -c \
	"cd /opt/ido-ewo/platform/cnf-vpp;make -C src install-dep;make -C src pkg"  
    _在guest中调用make命令，即VPP相关依赖安装和编译，生成pkg包,此步骤同时包含VPP集成组件的编译，如sweetcomb/RestAPI等_  
#### 内部make分析
	install-dep:
	sudo ${INSTALL} git wget patch
	make -C vpp $@
	make -C sweetcomb $@
	_分别进行VPP和Sweetcomb编译，如下为VPP编译，通过从FDIO官网下载指定版本，并打上功能补丁后编译_
	rm -rf vpp-$(VERSION) v$(VERSION).tar.gz
	git config --global user.name dummy
	git config --global user.email dummy@example.com
	./patch_vpp-$(VERSION).sh
	git add vpp-$(VERSION)/*
	git commit -a -m "To do the commit locally but not to push only as a workaround to make vpp build system happy"
	make -C vpp-$(VERSION) UNATTENDED=yes $@

3. cp ${CNFVPPROOT}/src/sweetcomb/${PKG}/*.${PKG} ${CNFVPPROOT}/src/vpp/vpp-20.09/build-root/*vpp*.${PKG} \
	${CNFVPPROOT}/src/sweetcomb/build-root/build/sweetcomb*.${PKG} ${CNFVPPROOT}/build/packages  
    _将编译结果拷贝至Host build/packages目录_  
### service-base 制作发布的docker应用（依赖于编译出的deb/rpm包）


# The official dCache image with the GSDC site patches applied on top.
#
# The build context must contain the files listed in overlay.txt, as
# prepared by .github/workflows/gsdc-build.yml.
#
# The official image ships a Java runtime made with jlink from the OpenJDK
# that AlmaLinux had when the image was built, and removes the RPM database,
# so the JDK cannot be updated with dnf. Instead, the runtime is made again
# here, the same way as upstream (packages/tar/src/main/container/Dockerfile),
# from the newest java-21-openjdk of AlmaLinux 10, the rebuild of Red Hat's
# OpenJDK with its security fixes.
ARG DCACHE_VERSION
ARG BUILDER_IMAGE=docker.io/library/almalinux:10-minimal

FROM ${BUILDER_IMAGE} AS jdk

# Empty: the newest java-21-openjdk in the repos of BUILDER_IMAGE. Otherwise
# the version (e.g. 21.0.13.1.2) or version-release to install; the build
# fails if the repos do not have it.
ARG OPENJDK_VERSION=

RUN microdnf -y --refresh install binutils \
        java-21-openjdk-devel${OPENJDK_VERSION:+-${OPENJDK_VERSION}} \
        java-21-openjdk-jmods${OPENJDK_VERSION:+-${OPENJDK_VERSION}} \
 && microdnf clean all

# Same jlink options and modules as upstream.
RUN jlink --compress=zip-6 --strip-debug --no-header-files --no-man-pages \
        --add-modules java.base,java.compiler,java.instrument,java.logging,java.management,java.naming,java.security.jgss,java.transaction.xa,java.xml,jdk.jfr,jdk.security.auth,jdk.unsupported,java.desktop,java.sql,jdk.naming.dns,java.rmi,jdk.crypto.cryptoki \
        --output /jlink-runtime \
 && rpm -q --qf '%{NAME}-%{VERSION}-%{RELEASE}\n' java-21-openjdk-headless > /jlink-runtime/OPENJDK_RPM \
 && /jlink-runtime/bin/java -version

FROM docker.io/dcache/dcache:${DCACHE_VERSION}

USER root
RUN rm -rf /jlink-runtime
COPY --from=jdk /jlink-runtime /jlink-runtime
USER dcache

COPY opt/dcache/ /opt/dcache/

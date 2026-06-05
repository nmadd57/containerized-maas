# Stage 1: Build patched squashfs-tools, repack snapd snap.
# Uses ubuntu:noble (full) for the compiler toolchain.
FROM ubuntu:noble AS snapd

ENV DEBIAN_FRONTEND=noninteractive
ENV LDFLAGS=-static
ENV PATH="$PATH:/snap/bin"

RUN apt-get update -qq && \
    apt-get dist-upgrade --yes && \
    apt-get install --yes -qq --no-install-recommends \
        fuse3 \
        gnupg \
        python3 \
        snapd \
        sudo \
        systemd \
        build-essential \
        git \
        help2man \
        zlib1g-dev \
        liblz4-dev \
        liblzma-dev \
        liblzo2-dev

RUN git clone https://github.com/plougher/squashfs-tools.git
RUN cd squashfs-tools && \
    git checkout 4.5.1 && \
    sed -Ei 's/#(XZ_SUPPORT.*)/\1/' squashfs-tools/Makefile && \
    sed -Ei 's/#(LZO_SUPPORT.*)/\1/' squashfs-tools/Makefile && \
    sed -Ei 's/#(LZ4_SUPPORT.*)/\1/' squashfs-tools/Makefile && \
    sed -Ei 's|(INSTALL_PREFIX = ).*|\1 /usr|' squashfs-tools/Makefile && \
    sed -Ei 's/\$\(INSTALL_DIR\)/$(DESTDIR)$(INSTALL_DIR)/g' squashfs-tools/Makefile && \
    cd squashfs-tools && \
    make -j$(nproc) && \
    make install

RUN mkdir -p /snap/snapd/current
RUN snap download snapd
RUN unsquashfs -f -d /snap/snapd/current snapd_*.snap

RUN cp /usr/bin/mksquashfs /snap/snapd/current/usr/bin
RUN cp /usr/bin/unsquashfs /snap/snapd/current/usr/bin

RUN mksquashfs /snap/snapd/current /snapd.snap


# Stage 2: Prepare the Noble base filesystem.
FROM ubuntu:noble AS base

ENV DEBIAN_FRONTEND=noninteractive
ENV PATH="$PATH:/snap/bin"

RUN apt-get update -qq && \
    apt-get dist-upgrade --yes && \
    apt-get install --yes -qq --no-install-recommends \
        fuse3 \
        gnupg \
        python3 \
        snapd \
        sudo \
        systemd

RUN apt-get clean
RUN rm -rf /var/lib/apt/lists

RUN touch /var/lib/snapd/system-key

# Noble uses merged-usr: /sbin -> /usr/sbin
RUN dpkg-divert --local --rename --add /usr/sbin/udevadm
RUN ln -s /bin/true /usr/sbin/udevadm

RUN rm -f \
        /etc/systemd/system/*.wants/* \
        /lib/systemd/system/local-fs.target.wants/* \
        /lib/systemd/system/multi-user.target.wants/* \
        /lib/systemd/system/sockets.target.wants/*initctl*

RUN find \
        /lib/systemd/system/sysinit.target.wants \
        \( -type f -or -type l \) -and -not -name '*systemd-tmpfiles-setup*' \
        -delete

RUN rm -f /lib/systemd/system/systemd-update-utmp-runlevel.service

# Disable /tmp automount (may live in either location on Noble)
RUN rm -vf \
        /usr/share/systemd/tmp.mount \
        /usr/lib/systemd/system/tmp.mount

RUN echo ShowStatus=no >> /etc/systemd/system.conf

RUN systemctl disable ondemand.service 2>/dev/null || true
RUN systemctl set-default basic.target

RUN systemctl enable snapd.service
RUN systemctl enable snapd.socket


# Final stage: assemble into a clean scratch image.
FROM scratch

ENV container=docker \
    init=/lib/systemd/systemd
ENV PATH="/snap/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

COPY --from=base / /
COPY --from=snapd /snapd.snap /snapd.snap

ADD entrypoint.sh /bin/
ADD systemd-detect-virt /usr/bin/

STOPSIGNAL SIGRTMIN+3

ENTRYPOINT ["/bin/entrypoint.sh"]
CMD ["maas"]

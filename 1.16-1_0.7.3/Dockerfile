ARG base_image="cgitest.azurecr.io/von-image:py36-1.16-1"

FROM ${base_image}

ARG agent_version=0.7.3
ARG agent_implementation="python"
ARG tag_name=aries-cloudagent
ARG tag_version=0.7.3
ARG acapy_reqs="[askar,bbs]"
ARG git_egg_ref

ENV SUMMARY="aries-cloudagent image" \
    DESCRIPTION="aries-cloudagent provides a base image for running Hyperledger Aries agents in Docker. \
    This image layers the $agent_implementation implementation of aries-cloudagent $agent_version \
    on top of the base image $base_image."

LABEL summary="$SUMMARY" \
    description="$DESCRIPTION" \
    io.k8s.description="$DESCRIPTION" \
    io.k8s.display-name="aries-cloudagent $tag_version" \
    name="$tag_name" \
    version="$tag_version" \
    maintainer=""

USER root
# Create default aca-py home (ACAPY_HOME) directory and set permissions.
# This is used by indy-vdr
RUN mkdir -p $HOME/.aries_cloudagent

# The root group needs access the directories under $HOME/.aries_cloudagent for the container to function in OpenShift.
RUN chown -R indy:root $HOME/.aries_cloudagent && \
    chmod -R ug+rw $HOME/.aries_cloudagent
USER indy

RUN pip install --no-cache-dir ${git_egg_ref}aries-cloudagent${acapy_reqs}==${agent_version}

ENTRYPOINT ["/bin/bash", "-c", "aca-py \"$@\"", "--"]

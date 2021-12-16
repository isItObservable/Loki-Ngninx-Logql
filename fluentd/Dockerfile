FROM fluent/fluentd-kubernetes-daemonset:v1.14.3-debian-forward-1.0

USER root
WORKDIR /

RUN gem install fluent-plugin-dynatrace -v 0.1.5
RUN gem install fluent-plugin-kubernetes_metadata_filter -v 2.9.1
RUN gem install fluent-plugin-multi-format-parser
RUN gem install fluent-plugin-concat
RUN gem install fluent-plugin-prometheus
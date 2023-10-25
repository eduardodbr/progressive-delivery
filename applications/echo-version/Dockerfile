FROM jeffail/benthos:4.22

COPY pipeline.yaml pipeline.yaml

ENTRYPOINT [ "/benthos" ]
CMD [ "-c", "pipeline.yaml" ]
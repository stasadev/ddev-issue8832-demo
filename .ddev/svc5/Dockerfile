FROM alpine:3.20
RUN echo "first stage" > /marker.txt

FROM busybox:1.36 AS alpine
RUN echo "I am svc5 (later stage named alpine)" > /marker.txt
CMD ["sleep", "infinity"]

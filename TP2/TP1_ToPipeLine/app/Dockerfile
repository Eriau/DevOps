# FROM # TODO: Choose a java JRE
# TODO: Add the compiled java (aka bytecode, aka .class)
# TODO: Run the Java with: “java Main” command.
FROM openjdk:11
COPY . /usr/src/myapp
WORKDIR /usr/src/myapp
RUN javac Main.java
FROM openjdk:11-jre
COPY --from=0 /usr/src/myapp/Main.class .
# Run java code with the JRE
CMD ["java", "Main"]

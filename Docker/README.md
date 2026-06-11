#### -p vs -P

"We use _**\-p (lowercase)**_ in local development environments and single-node production environments where we need a fixed, predictable entry point—like mapping port 80 to a web app so users can access it normally."

"We use _**\-P (uppercase)**_ in automated testing pipelines, CI/CD environments, or dynamic clusters. It allows multiple instances of the same image to spin up on a single runner machine simultaneously without throwing 'port already in use' errors, letting an external load balancer handle the traffic routing later."

#### Q. "What happens if two of these containers need to run on the same host machine, but they can't both use port 8088?"
Ans. "By designing the Dockerfile with an environment variable (ENV PRODUCT\_CATALOG\_PORT=8088), the image remains completely environment-agnostic. At runtime, I can inject a new port value using the -e flag without rebuilding the image. The Go binary dynamically reads this variable, allowing me to spin up multiple instances of the same image on different ports seamlessly."

#### Q. ARG vs ENV.
Ans. **ARG** variables are purely for build-time operations. They are completely unavailable once the container starts running. They are perfect for setting build configurations, package versions, or compile-time flags. 

**ENV** variables are built into the image metadata and persist into the runtime environment. They are designed to manage application state and configuration at runtime, allowing operators to change application behavior without recompiling or rebuilding the image.

#### Q. "Imagine you are reviewing a pull request from a junior developer. They want to avoid using ENV variables for security reasons, so they changed the Dockerfile to use an ARG instead. They declared ARG PORT=8080, used EXPOSE ${PORT}, and then built the image. Inside the Go code, they are using os.Getenv("PORT") to start the server. The build passes with zero errors, but when we run the container, it fails immediately or listens on the wrong port. Why is this happening, and how would you fix it while respecting their choice to use ARG instead of ENV?"
Ans. **Step 1: Diagnose the Root Cause (Why it fails)**: _The issue here is a fundamental misunderstanding of the Docker lifecycle._ _**ARG variables only exist during the docker build phase.**_ _Once the image layers are assembled, those ARG values completely evaporate and are omitted from the final runtime container environment.Because of this, when the Go binary boots up inside the running container and calls os.Getenv("PORT"), it receives an empty string. The application doesn't know what port to use, causing it to either crash or default to a system port, despite EXPOSE successfully documenting the port during the build phase._

**Step 2: Provide the Advanced Solution (Go Linker Flags)**:
_To fix this without resorting to runtime ENV variables, we need to bake the ARG value directly into the Go binary itself during compilation. We can achieve this by using_ _**Go Linker Flags (-ldflags)**_ _in the builder stage of our Dockerfile.First, in our Go code (main.go), we declare a package-level string variable instead of looking for an environment variable:_
```
package main

import "net/http"

// This variable will be targeted and overwritten at compile time
var port = "8080" 

func main() {
    http.ListenAndServe("0.0.0.0:" + port, nil)
}
```

**Step 3: Show the Modified Dockerfile Execution**:
Next, we update the Dockerfile to pass the ARG value straight into that Go variable during the compilation step using the -X flag:
```
FROM golang:1.22-alpine AS builder
WORKDIR /usr/src/app

# 1. Accept the build argument in the builder stage
ARG PRODUCT_CATALOG_PORT=8088

COPY go.mod go.sum ./
RUN go mod download
COPY . .

# 2. Compile-time injection: Override the 'port' variable inside 'main'
RUN go build -ldflags "-X main.port=${PRODUCT_CATALOG_PORT}" -o product-catalog ./

FROM alpine AS release
WORKDIR /usr/src/app
COPY --from=builder /usr/src/app/product-catalog ./

# 3. Re-declare ARG in the final stage so EXPOSE can read it
ARG PRODUCT_CATALOG_PORT=8088
EXPOSE ${PRODUCT_CATALOG_PORT}

ENTRYPOINT ["./product-catalog"]
```

By compiling it this way, the port number is hardcoded directly inside the compiled machine code of the Go binary. The application doesn't need to look for any external environment variables at runtime, completely solving the junior developer's problem while keeping the configuration secure and immutable.

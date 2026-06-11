#### -p vs -P

"We use _**\-p (lowercase)**_ in local development environments and single-node production environments where we need a fixed, predictable entry point—like mapping port 80 to a web app so users can access it normally."

"We use _**\-P (uppercase)**_ in automated testing pipelines, CI/CD environments, or dynamic clusters. It allows multiple instances of the same image to spin up on a single runner machine simultaneously without throwing 'port already in use' errors, letting an external load balancer handle the traffic routing later."

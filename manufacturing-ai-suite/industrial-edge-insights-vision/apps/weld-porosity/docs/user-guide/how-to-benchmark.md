# How to Run Benchmarking

This guide provides step-by-step instructions for running the benchmarking script to evaluate the performance of the Weld Porosity Detection application. The script can help you determine the maximum number of concurrent video streams your system can handle while meeting a specific performance target (e.g., frames per second).

## Overview of the Benchmarking Script

The `benchmark_start.sh` script, located in the `manufacturing-ai-suite/industrial-edge-insights-vision` directory, automates the process of running performance tests on the DL Streamer Pipeline Server. It offers two primary modes of operation:

*   **Fixed Stream Mode:** Runs a specified number of concurrent video processing pipelines. This mode is useful for testing a specific, known workload.
*   **Stream-Density Mode:** Automatically determines the maximum number of streams that can be processed while maintaining a target Frames Per Second (FPS). This is ideal for capacity planning and finding the performance limits of your hardware.

## Prerequisites

Before running the benchmarking script, ensure you have the following:

*   A successful deployment of the Weld Porosity Detection application using Docker Compose or Helm, as described in the [Get Started](./get-started.md) guide.
*   The `jq` command-line JSON processor must be installed. You can install it on Ubuntu with:
    ```bash
    sudo apt-get update && sudo apt-get install -y jq
    ```
*   The `bc` calculator for floating-point arithmetic must be installed:
    ```bash
    sudo apt-get install -y bc
    ```
*   The `benchmark_start.sh` script and payload configuration files must be available on your system.

## Understanding the Payload File

The benchmarking script requires a JSON payload file to configure the pipelines that will be tested. These payload files are located within the `apps/weld-porosity/` directory. The script uses this file to specify the pipeline to run and the configuration for the video source, destination, and parameters.

Here is an example of a payload file, `benchmark_gpu_payload.json`:

```json
[
    {
        "pipeline": "weld_porosity_detection_benchmarking",
        "payload": {
            "parameters": {
                "detection-properties": {
                    "model": "/home/pipeline-server/resources/models/weld-porosity/deployment/Detection/model/model.xml",
                    "device": "GPU",
                    "batch-size": 8,
                    "model-instance-id": "instgpu0",
                    "inference-interval": 3,
                    "inference-region": 0,
                    "nireq": 2,
                    "ie-config": "NUM_STREAMS=2",
                    "pre-process-backend": "va-surface-sharing",
                    "threshold": 0.7
                }
            }
        }
    }
]
```

*   `pipeline`: The name of the pipeline to execute (e.g., `weld_porosity_detection_benchmarking`).
*   `payload`: An object containing the configuration for the pipeline instance.
    *   `parameters`: Allows you to set pipeline-specific parameters, such as the `device` (CPU, GPU, or AUTO) and other model-related properties including batch size, inference intervals, and OpenVINO™ configurations.

## Step 1: Configure the Benchmarking Script

Before running the script, you may need to adjust the `DLSPS_NODE_IP` variable within `benchmark_start.sh` if your DL Streamer Pipeline Server is not running on `localhost`.

```bash
# Edit the benchmark_start.sh script if needed
nano benchmark_start.sh
```

Change `DLSPS_NODE_IP="localhost"` to the correct IP address of the node where the service is exposed.

## Step 2: Run the Benchmarking Script

The script can be run in two different modes.

Navigate to the `manufacturing-ai-suite/industrial-edge-insights-vision` directory to run the script.

```bash
cd edge-ai-suites/manufacturing-ai-suite/industrial-edge-insights-vision/
```

### Fixed Stream Mode

In this mode, you specify the exact number of pipelines to run concurrently. This is useful for simulating a known workload.

**To run 4 pipelines simultaneously using the GPU payload:**

```bash
./benchmark_start.sh -p apps/weld-porosity/benchmark_gpu_payload.json -n 4
```

*   `-p apps/weld-porosity/benchmark_gpu_payload.json`: Specifies the path to your payload configuration file.
*   `-n 4`: Sets the number of concurrent pipelines to run.

The script will start the 4 pipelines and print their status. You can then monitor their performance via Grafana at `https://localhost/grafana`, by using the pipeline status API endpoint, or by checking the FPS directly:

```bash
curl -k https://localhost/api/pipelines/status
```

### Stream-Density Mode

In this mode, the script automatically finds the maximum number of streams that can run while maintaining a target FPS. This is useful for determining the capacity of your system.

**To find the maximum number of streams that can achieve at least 28.5 FPS on GPU:**

```bash
./benchmark_start.sh -p apps/weld-porosity/benchmark_gpu_payload.json -t 28.5
```

*   `-p apps/weld-porosity/benchmark_gpu_payload.json`: Specifies the path to your payload configuration file.
*   `-t 28.5`: Sets the target average FPS per stream. The default is `28.5`.
*   `-i 60`: (Optional) Sets the monitoring interval in seconds for collecting FPS data. The default is `60`.

The script will start with one stream, measure the FPS, and if the target is met, it will stop, add another stream, and repeat the process. This continues until the average FPS drops below the target. The script will then report the maximum number of streams that successfully met the performance goal.

**Example Output:**

```
======================================================
✅ FINAL RESULT: Stream-Density Benchmark Completed!
   Maximum 3 stream(s) can achieve the target FPS of 28.5.
   
   Average FPS per stream for the optimal configuration:
     - Stream 1: 29.2 FPS
     - Stream 2: 28.8 FPS
     - Stream 3: 28.6 FPS
======================================================
```

### How Stream Performance is Evaluated

In Stream-Density Mode, the script evaluates if the system can sustain a target FPS across all concurrent streams. The process is as follows:

1.  **Individual Stream Monitoring:** The script monitors each running pipeline instance (stream) independently.
2.  **Sampling:** For the duration of the monitoring interval (e.g., 60 seconds), it samples the `avg_fps` value from each stream every 2 seconds.
3.  **Averaging per Stream:** After the interval, it calculates the average FPS for *each stream* based on the samples collected for that specific stream.
4.  **Validation:** The performance goal is considered met only if **every single stream's** calculated average FPS is greater than or equal to the target FPS. If even one stream falls below the target, the test fails for that number of concurrent streams.

This ensures that the reported optimal stream count represents a stable configuration where all streams are performing adequately, rather than relying on a combined average that could hide underperforming streams.

## Step 3: Stop the Benchmarking

The benchmarking script automatically stops all pipelines when running in Stream-Density mode. However, if you're running in Fixed Stream mode or need to manually stop pipelines, you can stop all running pipelines by interrupting the script with `Ctrl+C` or by running:

```bash
curl -k -X DELETE https://localhost/api/pipelines
```

Alternatively, you can stop individual pipelines by their ID:

```bash
# Get pipeline IDs
curl -k https://localhost/api/pipelines/status

# Stop specific pipeline
curl -k -X DELETE https://localhost/api/pipelines/<pipeline_id>
```

## Performance Optimization Tips

### GPU Optimization

*   **Batch Size:** Increase `batch-size` for better GPU utilization, but be mindful of memory constraints.
*   **Parallel Inference:** Tune `nireq` parameter to match your GPU's parallel processing capabilities.
*   **Stream Configuration:** Adjust `NUM_STREAMS` in `ie-config` to optimize for your specific GPU model.

### CPU Optimization

*   **Inference Interval:** Increase `inference-interval` to reduce CPU load if real-time processing isn't critical.
*   **Device Selection:** Use `device: "AUTO"` to let OpenVINO™ automatically select the best device.

### Memory Optimization

*   **Pre-process Backend:** Use `va-surface-sharing` for GPU memory efficiency.
*   **Model Precision:** Consider using INT8 or FP16 model precision for better performance.

## Summary

In this guide, you learned how to use the `benchmark_start.sh` script to run performance tests on your Weld Porosity Detection application. You can now measure performance for a fixed number of streams or automatically determine the maximum stream density your system can support.

Key takeaways:
*   Use Fixed Stream Mode for testing known workloads
*   Use Stream-Density Mode for capacity planning and finding system limits
*   Monitor individual stream performance to ensure consistent quality
*   Optimize payload configurations based on your hardware capabilities
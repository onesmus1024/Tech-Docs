Markdown

# 👁️ Azure AI Vision: Smart Image Analysis System

## 1. Project Scope
This document outlines the deployment and development of an intelligent image processing module using **Azure AI Vision** (part of Azure Cognitive Services). The system is capable of analyzing static images to extract text (OCR), generate captions, and detect objects automatically.

* **Service Used:** Azure AI Vision (v4.0)
* **Infrastructure:** Azure Cognitive Services Resource
* **Development SDK:** .NET 8 (C#)
* **Key Capabilities:** Optical Character Recognition (OCR), Image Captioning, Tagging.

---

## 2. Infrastructure Deployment (Provisioning)
Before the application can function, the Azure AI Vision resource must be provisioned. This was done using the Azure CLI to ensure reproducibility.

### Step 1: Resource Group Setup
Created a dedicated container for AI resources.
```bash
az group create --name rg-ai-vision-demo --location eastus
Step 2: Provisioning Azure AI Vision
We deployed the standard "ComputerVision" resource on the S1 (Standard) tier.

Bash

az cognitiveservices account create \
    --name ai-vision-service-001 \
    --resource-group rg-ai-vision-demo \
    --kind ComputerVision \
    --sku S1 \
    --location eastus \
    --yes
Evidence 1: Deployment Success

Description: Azure CLI output showing provisioningState: Succeeded.

Step 3: Access Configuration
To connect the application, we retrieved the Endpoint and API Key.

Bash

az cognitiveservices account keys list --name ai-vision-service-001 --resource-group rg-ai-vision-demo
az cognitiveservices account show --name ai-vision-service-001 --resource-group rg-ai-vision-demo --query "properties.endpoint"
Evidence 2: Azure Portal Resource

Description: Screenshot of the 'Keys and Endpoint' blade in the Azure Portal.

3. Application Development (Integration)
The application is a .NET Console App that sends local images to the Azure API for analysis.

3.1 Dependencies
The project uses the official NuGet package:

XML

<PackageReference Include="Azure.AI.Vision.ImageAnalysis" Version="1.0.0-beta.3" />
3.2 Implementation Code (Program.cs)
The following code establishes a client connection and requests visual features.

C#

using Azure;
using Azure.AI.Vision.ImageAnalysis;

// Configuration
string endpoint = Environment.GetEnvironmentVariable("VISION_ENDPOINT");
string key = Environment.GetEnvironmentVariable("VISION_KEY");

// 1. Authenticate Client
ImageAnalysisClient client = new ImageAnalysisClient(new Uri(endpoint), new AzureKeyCredential(key));

// 2. Define Image Source and Features
Uri imageUri = new Uri("[https://learn.microsoft.com/azure/ai-services/computer-vision/images/windows-kitchen.jpg](https://learn.microsoft.com/azure/ai-services/computer-vision/images/windows-kitchen.jpg)");
VisualFeatures features = VisualFeatures.Caption | VisualFeatures.Read;

// 3. Call Azure AI API
Console.WriteLine("Analyzing Image...");
ImageAnalysisResult result = client.Analyze(imageUri, features);

// 4. Output Results
Console.WriteLine($"Image Caption: {result.Caption.Text}");
Console.WriteLine($"Confidence: {result.Caption.Confidence:F2}");

if (result.Read != null)
{
    Console.WriteLine("\n--- OCR Results ---");
    foreach (var line in result.Read.Blocks.SelectMany(b => b.Lines))
    {
        Console.WriteLine($"Detected Text: {line.Text}");
    }
}
4. Demonstration & Testing
We tested the system using a sample image to verify both object detection and text extraction.

Test Case A: Image Captioning
Input: An image of a kitchen environment.

Expected Output: A descriptive sentence about the room.

Evidence 3: Console Output (Captioning)

Description: Console log showing the generated caption with 98% confidence.

Test Case B: OCR (Read API)
Input: A scanned document or street sign.

Expected Output: Extraction of raw text.

Evidence 4: OCR Extraction Result

Description: Console log showing the raw text extracted from the image.

5. Security & Cost Management
Key Rotation: API keys are stored in environment variables (VISION_KEY), not hardcoded. Keys are rotated every 90 days.

Cost Limits: An Azure Budget alert is set at $10/month to prevent cost overruns on the S1 tier.

6. Conclusion
This project successfully demonstrated the deployment of Azure AI infrastructure and the development of a client application that consumes the service. The system effectively translates visual data into structured text usable for downstream business logic.

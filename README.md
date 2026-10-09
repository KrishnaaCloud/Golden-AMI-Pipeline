# Enterprise Golden AMI Pipeline
Centralized Golden AMI pipeline using AWS EC2 Image Builder, distributing hardened, standardized AMIs across 9 enterprise AWS accounts.
## Architecture & Features
- **Image Baking**: AWS EC2 Image Builder
- **Distribution**: AWS Systems Manager (SSM) Parameter Store
- **Automation**: AWS EventBridge & Lambda for cross-account sharing
- **Observability**: Elastic Agent / Fleet
- **Infrastructure as Code**: AWS CDK (Python)

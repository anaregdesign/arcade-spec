# Execution Plan

- Spec: `/docs/spec/specification.md`
- README target: `/README.md`
- Diagram target: `/docs/diagrams/architecture-overview.svg`

## Section 1 - VNet Topology Refinement

- [x] Reconfirm the exact Azure network topology from `infra/main.bicep`, especially delegated and private-endpoint subnets.
- [x] Update the architecture SVG so the VNet boundary, subnet roles, and Private Link path match the current Bicep contract.
- [x] Adjust the README explanation if the diagram semantics change.
- [x] Validate the touched files and archive this completed plan.
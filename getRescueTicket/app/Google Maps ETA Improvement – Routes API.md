Google Maps ETA Improvement – Routes API Integration

You're right about the ETA discrepancies. We've identified the cause and confirmed that moving to Google's Routes API with computeRoutes is the right solution.

Currently, our system uses the Distance Matrix service without specifying a departure time. As a result, Google returns estimated travel times based on free-flow conditions, without accounting for traffic. The distances remain accurate because traffic does not affect the calculated distance.

Before we begin implementation, there are two things to confirm.

1. What Needs to Be Enabled on Your Side

One clarification: TRAFFIC_AWARE_OPTIMAL is not a setting in the Google Cloud Console. It is a parameter we pass with each API request.

Please complete the following steps:

Open Google Cloud Console → APIs & Services → Library.

Search for Routes API and click Enable.

Confirm that billing is active for the project.

Check the restrictions on your existing Google Maps API key. If it is restricted to specific APIs, add Routes API to the allowed list.

Important: These changes must be made in the same Google Cloud project that owns the existing API key. Otherwise, we would need to generate and configure a new key across all four apps.

2. Cost Implications

Traffic-aware routing uses Google's higher-priced Routes: Compute Routes Advanced billing tier, compared with the Distance Matrix pricing currently used. This increases the cost per request, so we want to make the impact clear before proceeding, especially since Google API usage was deliberately scoped to control costs.

However, there is a potential cost offset:

Current approach: Each calculation requests a full distance matrix, even though the system uses only part of the returned data.

Proposed approach: Routes API returns the specific route legs we need in a single request, reducing unnecessary results.

The system currently makes approximately 4–6 Google API calls per ticket under either approach. We can also calculate the expected cost based on your actual ticket volume to help you evaluate the change before approving the implementation.


# Welcome!

Contents hopefully coming really soon.
If things are not actually as documented here, please draft a pull request.

## Conceptual overview

To motivate hh200, let's mentally execute an HTTP server test scenario three times.
The test scenario says that a sequence of `POST /login { "username": Person, password }`
and `GET /status` (bearing login token) always return Person's status. We're also interested in learning
the number of parallel users after which the system-under-test starts to respond in >1 second.

### Alternatives

!!! abstract "Postman"
    Units of HTTP request are organized in "Collections". Unless we're in a team
    with a specific workflow, we can do the following:

    * In example-based manner, create requests for the POST and GET endpoints under a Collection.
    * **(Alternative 1)** Utilize pre-request and post-response "Scripts" to encode the necessary chaining effects (i.e. parsing access token, passing the token as variable)
    * **(Alternative 2)** Utilize "Flows". To my knowledge, at the time of writing, simulating parallel users is tricky even with this premium feature.

    We had AI sketch what the script could look like. The script does more (i.e. {average, max} {login, get status} time) than it's asked while it's at it, but after deleting
    the extraneous lines:

    <details>
        <summary>AI-generated pre-request & post-response scripts</summary>
        ```js linenums="1"
        // ===========================================
        // PRE-REQUEST SCRIPT FOR LOGIN ENDPOINT
        // ===========================================

        // Initialize or increment user counter for parallel testing
        let userCounter = pm.globals.get('userCounter') || 1;
        let currentIteration = pm.info.iteration + 1;
        let totalIterations = pm.info.iterationCount;

        // Generate unique username for this iteration
        const username = `user${userCounter}`;
        pm.collectionVariables.set('current_user', username);

        // Increment counter for next iteration
        pm.globals.set('userCounter', userCounter + 1);

        console.log(`Iteration ${currentIteration}/${totalIterations}: Testing with user ${username}`);

        // Initialize performance tracking
        if (!pm.globals.get('testResults')) {
            pm.globals.set('testResults', JSON.stringify([]));
        }

        // Set start time for this user's complete flow
        pm.globals.set(`startTime_${username}`, Date.now());

        // ===========================================
        // POST-RESPONSE SCRIPT FOR LOGIN ENDPOINT
        // ===========================================

        const username = pm.collectionVariables.get('current_user');
        const responseTime = pm.response.responseTime;

        // Test response status
        pm.test('Login successful', function () {
            pm.response.to.have.status(200);
        });

        // Test response time threshold
        pm.test('Login response time < 1s', function () {
            pm.expect(pm.response.responseTime).to.be.below(1000);
        });

        // Extract and save token
        if (pm.response.code === 200) {
            const response = pm.response.json();
            
            pm.test('Token exists in response', function () {
                pm.expect(response).to.have.property('token');
            });
            
            if (response.token) {
                pm.collectionVariables.set('auth_token', response.token);
                console.log(`Login successful for ${username} (${responseTime}ms)`);
            }
        } else {
            console.log(`Login failed for ${username}: Status ${pm.response.code}`);
        }

        // Track login performance
        pm.globals.set(`loginTime_${username}`, responseTime);

        // ===========================================
        // PRE-REQUEST SCRIPT FOR STATUS ENDPOINT
        // ===========================================

        const username = pm.collectionVariables.get('current_user');
        console.log(`Fetching data for ${username}`);

        // Ensure we have an auth token
        const token = pm.collectionVariables.get('auth_token');
        if (!token) {
            console.error(`No auth token available for ${username}`);
        }

        // ===========================================
        // POST-RESPONSE SCRIPT FOR STATUS ENDPOINT
        // ===========================================

        const username = pm.collectionVariables.get('current_user');
        const responseTime = pm.response.responseTime;
        const loginTime = pm.globals.get(`loginTime_${username}`) || 0;
        const startTime = pm.globals.get(`startTime_${username}`) || Date.now();
        const totalTime = Date.now() - startTime;

        // Test response status
        pm.test('Data retrieval successful', function () {
            pm.response.to.have.status(200);
        });

        // Test response time threshold
        pm.test('Data response time < 1s', function () {
            pm.expect(pm.response.responseTime).to.be.below(1000);
        });

        // Verify user data
        if (pm.response.code === 200) {
            const response = pm.response.json();
            
            pm.test('Correct user data returned', function () {
                pm.expect(response.username).to.equal(username);
            });
            
            console.log(`Data retrieval successful for ${username} (${responseTime}ms)`);
        } else {
            console.log(`Data retrieval failed for ${username}: Status ${pm.response.code}`);
        }

        // Collect performance data
        let testResults = JSON.parse(pm.globals.get('testResults') || '[]');
        testResults.push({
            username: username,
            loginTime: loginTime,
            dataTime: responseTime,
            totalTime: totalTime,
            loginSuccess: pm.globals.get(`loginTime_${username}`) ? true : false,
            dataSuccess: pm.response.code === 200,
            timestamp: new Date().toISOString()
        });

        pm.globals.set('testResults', JSON.stringify(testResults));

        // Performance analysis on final iteration
        const currentIteration = pm.info.iteration + 1;
        const totalIterations = pm.info.iterationCount;

        if (currentIteration === totalIterations) {
            console.log('\n=== PERFORMANCE ANALYSIS ===');
            
            const results = JSON.parse(pm.globals.get('testResults') || '[]');
            const successfulResults = results.filter(r => r.loginSuccess && r.dataSuccess);
            const failedResults = results.filter(r => !r.loginSuccess || !r.dataSuccess);
            
            // Calculate statistics
            const totalTests = results.length;
            const successCount = successfulResults.length;
            const failureCount = failedResults.length;
            const successRate = (successCount / totalTests * 100).toFixed(1);
            
            // Response time analysis
            const loginTimes = successfulResults.map(r => r.loginTime);
            const dataTimes = successfulResults.map(r => r.dataTime);
            const totalTimes = successfulResults.map(r => r.totalTime);
            
            // Count slow responses (>1s)
            const slowLoginResponses = loginTimes.filter(t => t > 1000).length;
            const slowDataResponses = dataTimes.filter(t => t > 1000).length;
            const totalSlowResponses = slowLoginResponses + slowDataResponses;
            
            console.log(`Total Users Tested: ${totalTests}`);
            console.log(`Successful: ${successCount} (${successRate}%)`);
            console.log(`Failed: ${failureCount}`);
            console.log(`Slow Responses (>1s): ${totalSlowResponses}`);
            console.log(`  - Slow Login: ${slowLoginResponses}`);
            console.log(`  - Slow Data: ${slowDataResponses}`);
            
            // Threshold detection
            if (totalSlowResponses > 0) {
                console.log(`\n*** PERFORMANCE THRESHOLD DETECTED ***`);
                console.log(`System shows degraded performance with ${totalTests} parallel users`);
                console.log(`${totalSlowResponses} responses exceeded 1 second threshold`);
            } else {
                console.log(`\n*** ALL RESPONSES UNDER 1 SECOND ***`);
                console.log(`System handles ${totalTests} parallel users efficiently`);
                console.log(`Consider testing with more users to find threshold`);
            }
            
            // Performance test results for collection variables
            pm.collectionVariables.set('final_results', JSON.stringify({
                totalUsers: totalTests,
                successRate: successRate,
                slowResponses: totalSlowResponses,
                thresholdExceeded: totalSlowResponses > 0
            }));
            
            // Clean up globals
            pm.globals.unset('userCounter');
            pm.globals.unset('testResults');
            
            // Clean up individual user data
            for (let i = 1; i <= totalTests; i++) {
                pm.globals.unset(`startTime_user${i}`);
                pm.globals.unset(`loginTime_user${i}`);
            }
        }

        // ===========================================
        // COLLECTION-LEVEL PRE-REQUEST SCRIPT
        // ===========================================

        // Initialize test environment
        console.log('Initializing Login Performance Load Test...');

        // Reset any previous test data
        pm.globals.unset('userCounter');
        pm.globals.unset('testResults');

        // Set test configuration
        const testConfig = {
            baseUrl: pm.collectionVariables.get('base_url') || 'http://staging.example.com',
            responseTimeThreshold: parseInt(pm.collectionVariables.get('response_time_threshold') || '1000'),
            maxUsers: 50
        };

        console.log('Test Configuration:', testConfig);
        console.log(`Testing will run with ${pm.info.iterationCount} parallel users`);
        console.log(`Looking for responses exceeding ${testConfig.responseTimeThreshold}ms threshold`);

        // ===========================================
        // COLLECTION-LEVEL POST-REQUEST SCRIPT
        // ===========================================

        // This runs after all requests in the collection
        console.log('Load test execution completed');

        // Final cleanup and summary
        const finalResults = pm.collectionVariables.get('final_results');
        if (finalResults) {
            const results = JSON.parse(finalResults);
            console.log('\n=== FINAL SUMMARY ===');
            console.log(`Tested ${results.totalUsers} parallel users`);
            console.log(`Success Rate: ${results.successRate}%`);
            console.log(`Performance Threshold ${results.thresholdExceeded ? 'EXCEEDED' : 'NOT REACHED'}`);
            
            if (results.thresholdExceeded) {
                console.log(`⚠️ System performance degrades with ${results.totalUsers} parallel users`);
            } else {
                console.log(`✅ System handles ${results.totalUsers} parallel users efficiently`);
            }
        }
        ```
    </details>

!!! abstract "Hurl"
    Without specifying the Hurl version to keep things conceptual, the main bit can look as follows for the simplest case of testing one user.
    ```sh linenums="1"
    POST http://staging.example.com/login
    Content-Type: application/json
    {
      "username": "bob",
      "password": "loremipsum"
    }
    HTTP 200
    [Captures]
    AUTH_TOKEN: jsonpath "$.token"

    GET http://staging.example.com/status
    Authorization: Bearer {{AUTH_TOKEN}}
    HTTP 200
    [Asserts]
    jsonpath "$.username" == "bob"
    duration < 1000
    ```
    We can express parallel users with OS processes (using bash script). In the following script, one user gets one unix PID.
    <details>
        <summary>AI-generated final bash script</summary>
        ```sh linenums="1" hl_lines="51-56"
        #!/bin/bash

        # Parallel load test script to find performance threshold
        # This script tests increasing numbers of parallel users

        BASE_URL="http://staging.example.com"
        MAX_USERS=50
        STEP=5

        echo "Testing parallel user performance threshold..."
        echo "Looking for the point where response time exceeds 1 second"
        echo "========================================================="

        # Create individual user test files
        create_user_test() {
            local user_num=$1
            cat > "/tmp/user_${user_num}.hurl" << EOF
        POST ${BASE_URL}/login
        Content-Type: application/json
        {
            "username": "user${user_num}",
            "password": "password123"
        }

        HTTP 200
        [Captures]
        AUTH_TOKEN: jsonpath "$.token"

        GET ${BASE_URL}/status
        Authorization: Bearer {{AUTH_TOKEN}}

        HTTP 200
        [Asserts]
        jsonpath "$.username" == "user${user_num}"
        duration < 1000
        EOF
        }

        # Test with increasing number of parallel users
        for users in $(seq $STEP $STEP $MAX_USERS); do
            echo "Testing with $users parallel users..."
            
            # Create test files for each user
            for i in $(seq 1 $users); do
                create_user_test $i
            done
            
            # Start timestamp
            start_time=$(date +%s.%N)
            
            # Run parallel tests
            pids=()
            for i in $(seq 1 $users); do
                hurl --test --very-verbose "/tmp/user_${i}.hurl" > "/tmp/result_${i}.log" 2>&1 &
                pids+=($!)
            done
            
            # Wait for all tests to complete
            failed_count=0
            slow_count=0
            
            for pid in "${pids[@]}"; do
                if ! wait $pid; then
                    ((failed_count++))
                fi
            done
            
            # Check results for slow responses
            for i in $(seq 1 $users); do
                if grep -q "duration < 1000.*false" "/tmp/result_${i}.log" 2>/dev/null; then
                    ((slow_count++))
                fi
            done
            
            end_time=$(date +%s.%N)
            total_time=$(echo "$end_time - $start_time" | bc)
            
            echo "Results for $users users:"
            echo "  - Total test time: ${total_time}s"
            echo "  - Failed tests: $failed_count"
            echo "  - Slow responses (>1s): $slow_count"
            
            # If we have slow responses, we've found our threshold
            if [ $slow_count -gt 0 ]; then
                echo "*** THRESHOLD FOUND: System starts responding >1s with $users parallel users ***"
                echo "*** Slow responses detected: $slow_count out of $users ***"
                break
            fi
            
            # Clean up temp files
            rm -f /tmp/user_*.hurl /tmp/result_*.log
            
            echo "All responses under 1s with $users users, continuing..."
            echo ""
            
            # Brief pause between test runs
            sleep 2
        done

        echo "Load test completed."
        ```
    </details>

!!! abstract "General purpose language"
    Why not a general purpose language like python? I've been developing an unreleased python package that allows the following program.

    ```python linenums="1"
    import pp200

    ...
    ```

    Alternatively, the following looks good to me. (Truly, unironically. The point of this section is to visualize the accidental complexity of using a GPL.)
    <details>
        <summary>AI-generated alternative python implementation</summary>
        ```python linenums="1"
        """
        Login Performance Load Test
        Tests the login/status retrieval flow and finds the threshold where response time exceeds 1 second
        """

        import asyncio
        import aiohttp
        import time
        import json
        from typing import Dict, List, Tuple
        from dataclasses import dataclass

        @dataclass
        class TestResult:
            user_id: str
            login_time: float
            data_time: float
            success: bool
            error: str = ""

        class LoginLoadTester:
            def __init__(self, base_url: str = "http://staging.example.com"):
                self.base_url = base_url
                self.results: List[TestResult] = []
                
            async def test_single_user(self, session: aiohttp.ClientSession, user_num: int) -> TestResult:
                """Test login and data retrieval for a single user using async"""
                username = f"user{user_num}"
                
                try:
                    # Login request
                    login_start = time.time()
                    login_payload = {
                        "username": username,
                        "password": "password123"
                    }
                    
                    async with session.post(
                        f"{self.base_url}/login",
                        json=login_payload,
                        headers={"Content-Type": "application/json"}
                    ) as login_response:
                        login_end = time.time()
                        login_time = login_end - login_start
                        
                        if login_response.status != 200:
                            return TestResult(username, login_time, 0, False, f"Login failed: {login_response.status}")
                        
                        login_data = await login_response.json()
                        token = login_data.get("token")
                        
                        if not token:
                            return TestResult(username, login_time, 0, False, "No token in login response")
                    
                    # Data retrieval request
                    data_start = time.time()
                    async with session.get(
                        f"{self.base_url}/status",
                        headers={"Authorization": f"Bearer {token}"}
                    ) as data_response:
                        data_end = time.time()
                        data_time = data_end - data_start
                        
                        if data_response.status != 200:
                            return TestResult(username, login_time, data_time, False, f"Data request failed: {data_response.status}")
                        
                        data = await data_response.json()
                        
                        # Verify the response contains the correct user data
                        if data.get("username") != username:
                            return TestResult(username, login_time, data_time, False, "Username mismatch in response")
                        
                        return TestResult(username, login_time, data_time, True)
                        
                except Exception as e:
                    return TestResult(username, 0, 0, False, str(e))
            

            
            async def run_load_test(self, num_users: int) -> Tuple[List[TestResult], float]:
                """Run load test with specified number of parallel users using async"""
                start_time = time.time()
                
                connector = aiohttp.TCPConnector(limit=None, limit_per_host=None)
                timeout = aiohttp.ClientTimeout(total=60)
                
                async with aiohttp.ClientSession(connector=connector, timeout=timeout) as session:
                    tasks = [self.test_single_user(session, i) for i in range(1, num_users + 1)]
                    results = await asyncio.gather(*tasks, return_exceptions=True)
                    
                    # Handle any exceptions that occurred
                    processed_results = []
                    for i, result in enumerate(results):
                        if isinstance(result, Exception):
                            processed_results.append(TestResult(f"user{i+1}", 0, 0, False, str(result)))
                        else:
                            processed_results.append(result)
                
                end_time = time.time()
                total_time = end_time - start_time
                
                return processed_results, total_time
            

            
            def analyze_results(self, results: List[TestResult]) -> Dict:
                """Analyze test results and return statistics"""
                successful_results = [r for r in results if r.success]
                failed_results = [r for r in results if not r.success]
                
                if not successful_results:
                    return {
                        "total_tests": len(results),
                        "successful": 0,
                        "failed": len(failed_results),
                        "success_rate": 0.0,
                        "avg_login_time": 0,
                        "avg_data_time": 0,
                        "max_login_time": 0,
                        "max_data_time": 0,
                        "slow_responses": 0,
                        "errors": [r.error for r in failed_results]
                    }
                
                login_times = [r.login_time for r in successful_results]
                data_times = [r.data_time for r in successful_results]
                
                # Count responses over 1 second
                slow_login = len([t for t in login_times if t > 1.0])
                slow_data = len([t for t in data_times if t > 1.0])
                slow_responses = slow_login + slow_data
                
                return {
                    "total_tests": len(results),
                    "successful": len(successful_results),
                    "failed": len(failed_results),
                    "success_rate": len(successful_results) / len(results) * 100,
                    "avg_login_time": sum(login_times) / len(login_times),
                    "avg_data_time": sum(data_times) / len(data_times),
                    "max_login_time": max(login_times),
                    "max_data_time": max(data_times),
                    "slow_responses": slow_responses,
                    "slow_login_responses": slow_login,
                    "slow_data_responses": slow_data,
                    "errors": list(set([r.error for r in failed_results if r.error]))
                }
            
            def find_performance_threshold(self, max_users: int = 50, step: int = 5):
                """Find the number of parallel users where response time exceeds 1 second"""
                print("Testing parallel user performance threshold...")
                print("Looking for the point where response time exceeds 1 second")
                print("=" * 60)
                
                threshold_found = False
                
                for num_users in range(step, max_users + 1, step):
                    print(f"\nTesting with {num_users} parallel users...")
                    
                    try:
                        results, total_time = asyncio.run(self.run_load_test(num_users))
                        
                        stats = self.analyze_results(results)
                        
                        print(f"Results for {num_users} users:")
                        print(f"  - Total test time: {total_time:.2f}s")
                        print(f"  - Success rate: {stats['success_rate']:.1f}%")
                        print(f"  - Successful tests: {stats['successful']}")
                        print(f"  - Failed tests: {stats['failed']}")
                        print(f"  - Avg login time: {stats['avg_login_time']:.3f}s")
                        print(f"  - Avg data time: {stats['avg_data_time']:.3f}s")
                        print(f"  - Max login time: {stats['max_login_time']:.3f}s")
                        print(f"  - Max data time: {stats['max_data_time']:.3f}s")
                        print(f"  - Slow responses (>1s): {stats['slow_responses']}")
                        
                        if stats['errors']:
                            print(f"  - Errors: {stats['errors'][:3]}...")  # Show first 3 errors
                        
                        # Check if we've found the threshold
                        if stats['slow_responses'] > 0 or stats['max_login_time'] > 1.0 or stats['max_data_time'] > 1.0:
                            print(f"\n*** THRESHOLD FOUND ***")
                            print(f"System starts responding >1s with {num_users} parallel users")
                            print(f"Slow login responses: {stats['slow_login_responses']}")
                            print(f"Slow data responses: {stats['slow_data_responses']}")
                            threshold_found = True
                            break
                        
                        print(f"All responses under 1s with {num_users} users, continuing...")
                        
                        # Brief pause between test runs
                        time.sleep(1)
                        
                    except Exception as e:
                        print(f"Error testing {num_users} users: {e}")
                        continue
                
                if not threshold_found:
                    print(f"\nNo threshold found up to {max_users} users. System handles load well!")
                
                print("\nLoad test completed.")

        def main():
            # Example usage
            tester = LoginLoadTester("http://staging.example.com")
            
            # Test basic functionality first
            print("Testing basic functionality...")
            result, _ = asyncio.run(tester.run_load_test(1))
            
            if result[0].success:
                print("✓ Basic functionality test passed")
                
                # Find performance threshold
                tester.find_performance_threshold(max_users=50, step=5)
            else:
                print(f"✗ Basic functionality test failed: {result[0].error}")
                print("Please check your server setup before running load tests")

        if __name__ == "__main__":
            main()
        ```
    </details>


The following table hopes to list a fair overview for the three methods.

| method [link] | slogan                                                                                       |
|---------------|----------------------------------------------------------------------------------------------|
| [Postman]     | Explore, test, and document APIs collaboratively in one place                                |
| [Hurl]        | Requests in simple text format.                                                              |
| (any GPL)     | The test scripts are as important as the system-under-test itself; might as well use the same main language. |

### Argued ultimate abstraction

Hopefully we have set the stage well for introducing hh200. This is one example of HTTP server testing problem,
the domain where hh200 tries to specialize in.

```sh linenums="1"
"get token"
POST http://staging.example.com/login
Content-Type: application/json
{
  "username": "bob",
  "password": "loremipsum"
}
HTTP 200
[Captures]
AUTH_TOKEN: jsonpath "$.token"

"get token" then "status with token"
GET http://staging.example.com/status
Authorization: Bearer {{AUTH_TOKEN}}
HTTP 200
[Asserts]
jsonpath "$.username" == "bob"
duration < 1000
```


All in all, hh200 doesn't compete with Postman GUI; agrees with
the above python approach where complexity is preferably hidden; and ultimately aspires to be more modern Hurl.

The following extensions to the above test scenario might help weighing whether
hh200 serves your taste decently well.

- insert another endpoint call after "get token"
- reorder the endpoint calls
- warn if "get token" returns 201 code instead of 200
- read from csv for all users that we want to serve in parallel
- specify the target number of parallel users before which it starts to respond with non-success codes
- download the status (`GET /status&file=xls`)
- download a file whose name already exist (keeping both)


## API reference
- <hackage project url\>
- <hoogle\>

[Postman]: https://www.postman.com/pricing
[Hurl]: https://hurl.dev

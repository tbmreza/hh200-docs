# Home

Welcome! Contents hopefully coming really soon.
If things are not actually as documented here, please draft a pull request.

## Conceptual overview

To motivate hh200, let's mentally execute an HTTP server test scenario three times.
The test scenario says that a sequence of `POST /login { "username": Person, password }`
and `GET /status` always return Person's report. We're also interested in learning
the number of parallel users after which the system-under-test starts to respond in >1 second.

!!! abstract "Postman"
    Units of HTTP request are organized in "Collections". Unless we're in a team
    with a specific workflow, we can do the following:

    * In example-based manner, create requests for the POST and GET endpoints under a Collection.
    * **(Alternative 1)** Utilize pre-request and post-response "Scripts" to encode the necessary chaining effects (i.e. parsing access token, passing the token as variable)
    * **(Alternative 2)** Utilize "Flows". To my knowledge, at the time of writing, simulating parallel users is tricky even with this premium feature.

!!! abstract "Hurl"
    [Hurl](https://hurl.dev) version unspecified to keep things conceptual, but the main bit can look as follows for the simplest case of testing one user.
    ```sh
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
        ```sh
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

        import pp200

        ...

    For the convenience of the reader:
    <details>
        <summary>AI-generated alternative python implementation</summary>
        ```python
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


No one method, including hh200, is superior most of the time in all situations. I'll leave with the following summary (as they seem to me)
of each method:

| method        | slogan                                                                                       |
|---------------|----------------------------------------------------------------------------------------------|
| Postman       | Explore, test, and document APIs collaboratively in one place                                |
| Hurl          | Requests in simple text format.                                                              |
| Python (or any GPL) | The test scripts are as important as the system-under-test itself; might as well use the same main language. |


All in all, hh200 doesn't compete with Postman GUI; agrees with
the above python approach where complexity is preferably hidden; aspires to be more modern Hurl.


## API reference
- <hackage project url\>
- <hoogle\>

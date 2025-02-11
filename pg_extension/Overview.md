This PostgreSQL extension integrates with the PostgreSQL query planner and executor to optimize query plans using an external service called Bao. The extension hooks into various stages of query processing to interact with the Bao server, collect query execution statistics, and provide optimization hints. A detailed explanation of the flow:

### Initialization

1. **Module Initialization (`_PG_init`)**:
   - The `_PG_init` function is called when the extension is loaded.
   - It installs hooks for the planner, executor start, executor end, and EXPLAIN functions.
   - It defines several custom configuration variables that control the behavior of the Bao optimizer.

### Planning Phase

2. **Planner Hook (`bao_planner`)**:
   - The `bao_planner` function is called before the PostgreSQL optimizer handles a query.
   - It checks if Bao optimization is enabled and if the query should be optimized by Bao.
   - If Bao optimization is enabled, it calls the `plan_query` function to get a query plan from the Bao server.
   - The `plan_query` function:
     - Connects to the Bao server.
     - Generates query plans for different "arms" (planner configurations).
     - Sends these plans to the Bao server and receives a selected plan.
   - The selected plan is returned and associated with the query using the `queryId` field.

### Execution Phase

3. **Executor Start Hook (`bao_ExecutorStart`)**:
   - The `bao_ExecutorStart` function is called at the start of query execution.
   - It sets up timing instrumentation if Bao reward collection is enabled.
   - This timing information will be used later to calculate the reward for the query.

4. **Executor End Hook (`bao_ExecutorEnd`)**:
   - The `bao_ExecutorEnd` function is called at the end of query execution.
   - It checks if Bao reward collection is enabled and if the query was optimized by Bao.
   - If so, it connects to the Bao server and sends the query execution time as a reward.
   - It extracts the `BaoQueryInfo` from the `queryId` field and sends the query plan, buffer state, and reward to the Bao server.

### EXPLAIN Phase

5. **EXPLAIN Hook (`bao_ExplainOneQuery`)**:
   - The `bao_ExplainOneQuery` function is called when an EXPLAIN command is issued.
   - It adds Bao's prediction and recommended hints to the EXPLAIN output.
   - It connects to the Bao server to get an estimate of the query plan's execution time.
   - It also generates a query plan using Bao and includes the recommended hint in the EXPLAIN output.

### Cleanup

6. **Module Finalization (`_PG_fini`)**:
   - The `_PG_fini` function is called when the extension is unloaded.
   - It logs a message indicating that the extension has finished.

### Configuration Variables

- **`enable_bao`**: Enables or disables the Bao optimizer.
- **`enable_bao_rewards`**: Enables or disables reward collection for queries.
- **`enable_bao_selection`**: Enables or disables Bao query plan selection.
- **`bao_host`**: Specifies the Bao server host.
- **`bao_port`**: Specifies the Bao server port.
- **`bao_num_arms`**: Specifies the number of planner configurations to consider.
- **`bao_include_json_in_explain`**: Includes Bao's JSON representation in EXPLAIN output.

### Key Functions and Structures

- **`plan_query`**: Generates query plans for different arms and communicates with the Bao server to get the selected plan.
- **`BaoPlan`**: A structure containing a PostgreSQL plan and Bao-specific information.
- **`BaoQueryInfo`**: A structure containing JSON representations of the query plan and buffer state.
- **`free_bao_plan`**: Frees a `BaoPlan` structure.
- **`free_bao_query_info`**: Frees a `BaoQueryInfo` structure.
- **`should_report_reward`**: Determines if the reward for a query should be reported to the Bao server.
- **`connect_to_bao`**: Connects to the Bao server.
- **`write_all_to_socket`**: Writes data to a socket.
- **`reward_json`**: Generates a JSON representation of the query reward.
- **`plan_to_json`**: Converts a query plan to a JSON representation.
- **`buffer_state`**: Gets the buffer state as a JSON representation.
- **`arm_to_hint`**: Translates an arm index into SQL statements for EXPLAIN hints.

### Summary

The extension integrates with PostgreSQL's query processing pipeline to optimize query plans using the Bao server. It hooks into the planner, executor start, executor end, and EXPLAIN phases to interact with the Bao server, collect execution statistics, and provide optimization hints. The extension is controlled by several configuration variables that enable or disable different aspects of Bao's functionality.

---

## Data Movement:

### Query: 

- The extension sends following things to the server:
   - Plans for each of the 5 arms.
      - Plan costs and CardEsts are added by the standard planner of PG under the specific tuning of a given arm.
   - Buffer state:
   ```c
   // modified from pg_buffercache_pages
   static char* buffer_state() {
   // Generate a JSON string mapping each relation name to the number of buffered
   // blocks that relation has in the PG buffer cache.
   }
   ```

   - Refer [bao_bufferstate.h](./bao_bufferstate.h).

- The server returns:
   -  Index of the chosen plan.

## Reward:
   - Executed Plan
   - Reward
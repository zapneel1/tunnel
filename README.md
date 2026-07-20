package com.scoreme;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.JsonNode;
import java.io.File;
import java.util.*;

public class MCPSSolver {

    public static class Task {
        int id;
        double[] resources;
        int windowStart, windowEnd;
        double weight;

        public Task(int id, double[] resources, int windowStart, int windowEnd, double weight) {
            this.id = id;
            this.resources = resources;
            this.windowStart = windowStart;
            this.windowEnd = windowEnd;
            this.weight = weight;
        }

        public int getWindowSize() {
            return (windowEnd - windowStart) + 1;
        }
    }

    public static class Result {
        public Map<String, Integer> assignment = new HashMap<>();
        public double penalty = 0.0;
        public long runtime_ms = 0;
        public boolean feasible = false;
        public String violation_reason = "";
    }

    public static Result solveMCPS(int n, int K, List<int[]> conflicts, 
                                   double[][] resources, double[][] originalCapacities, 
                                   int[][] windows, double[] weights) {
        long startTime = System.currentTimeMillis();
        Result result = new Result();

        Task[] tasks = new Task[n];
        List<Integer>[] adjList = new ArrayList[n];
        
        for (int i = 0; i < n; i++) {
            tasks[i] = new Task(i, resources[i], windows[i][0], windows[i][1], weights[i]);
            adjList[i] = new ArrayList<>();
        }

        for (int[] edge : conflicts) {
            adjList[edge[0]].add(edge[1]);
            adjList[edge[1]].add(edge[0]);
        }

        double[][] currentCapacities = new double[K][4];
        for (int s = 0; s < K; s++) {
            System.arraycopy(originalCapacities[s], 0, currentCapacities[s], 0, 4);
        }

        Map<Integer, Integer> tempAssignment = new HashMap<>();
        
        // Use a PriorityQueue instead of sorting a List on every iteration
        PriorityQueue<Task> unassignedQueue = new PriorityQueue<>((t1, t2) -> {
            int size1 = t1.getWindowSize();
            int size2 = t2.getWindowSize();
            if (size1 != size2) {
                return Integer.compare(size1, size2);
            }
            return Double.compare(t2.weight, t1.weight);
        });

        Collections.addAll(unassignedQueue, tasks);

        // Keep track of forbidden slots per task dynamically derived from neighbor conflicts
        // To replace the heavy validSlots set modification loop
        Map<Integer, Set<Integer>> forbiddenSlots = new HashMap<>();
        for (int i = 0; i < n; i++) {
            forbiddenSlots.put(i, new HashSet<>());
        }

        while (!unassignedQueue.isEmpty()) {
            Task target = unassignedQueue.poll();
            Integer bestSlot = null;
            double minPenaltyIncrease = Double.MAX_VALUE;

            Set<Integer> forbidden = forbiddenSlots.get(target.id);

            for (int s = target.windowStart; s <= target.windowEnd; s++) {
                if (forbidden.contains(s)) continue;

                boolean hasConflict = false;
                for (int neighborId : adjList[target.id]) {
                    if (tempAssignment.getOrDefault(neighborId, -1) == s) {
                        hasConflict = true;
                        break;
                    }
                }
                if (hasConflict) continue;

                boolean capacityExceeded = false;
                for (int d = 0; d < 4; d++) {
                    if (target.resources[d] > currentCapacities[s][d]) {
                        capacityExceeded = true;
                        break;
                    }
                }
                if (capacityExceeded) continue;

                double pBase = target.weight * s;
                double slotOriginalGpu = originalCapacities[s][2];
                double slotUsedGpu = slotOriginalGpu - currentCapacities[s][2] + target.resources[2];
                double pFrag = 0.0;
                double lambda = 10.0; 
                
                if (slotUsedGpu > 0 && slotOriginalGpu > 0) {
                    pFrag = lambda * (1.0 - (slotUsedGpu / slotOriginalGpu));
                }

                double penaltyDelta = pBase + pFrag;

                if (penaltyDelta < minPenaltyIncrease) {
                    minPenaltyIncrease = penaltyDelta;
                    bestSlot = s;
                }
            }

            if (bestSlot == null) {
                result.runtime_ms = System.currentTimeMillis() - startTime;
                result.feasible = false;
                result.violation_reason = "Task T" + target.id + " failed optimization parameters.";
                return result;
            }

            tempAssignment.put(target.id, bestSlot);
            result.assignment.put("T" + target.id, bestSlot);
            result.penalty += (target.weight * bestSlot); 
            
            for (int d = 0; d < 4; d++) {
                currentCapacities[bestSlot][d] -= target.resources[d];
            }

            // Update forbidden slots for adjacent tasks instead of modifying a full set per task object
            for (int neighborId : adjList[target.id]) {
                forbiddenSlots.get(neighborId).add(bestSlot);
            }
        }

        result.feasible = true;
        result.runtime_ms = System.currentTimeMillis() - startTime;
        return result;
    }

    public static void main(String[] args) {
        if (args.length < 2) {
            System.err.println("Usage: MCPSSolver <input_json> <output_json>");
            System.exit(1);
        }

        try {
            ObjectMapper mapper = new ObjectMapper();
            JsonNode root = mapper.readTree(new File(args[0]));

            int n = root.get("n").asInt();
            int K = root.get("K").asInt();

            List<int[]> conflicts = new ArrayList<>();
            for (JsonNode edge : root.get("conflicts")) {
                conflicts.add(new int[]{edge.get(0).asInt(), edge.get(1).asInt()});
            }

            double[][] resources = new double[n][4];
            int idx = 0;
            for (JsonNode res : root.get("resources")) {
                for (int d = 0; d < 4; d++) resources[idx][d] = res.get(d).asDouble();
                idx++;
            }

            double[][] capacities = new double[K][4];
            idx = 0;
            for (JsonNode cap : root.get("capacities")) {
                for (int d = 0; d < 4; d++) capacities[idx][d] = cap.get(d).asDouble();
                idx++;
            }

            int[][] windows = new int[n][2];
            idx = 0;
            for (JsonNode win : root.get("windows")) {
                windows[idx][0] = win.get(0).asInt();
                windows[idx][1] = win.get(1).asInt();
                idx++;
            }

            double[] weights = new double[n];
            idx = 0;
            for (JsonNode w : root.get("weights")) {
                weights[idx] = w.asDouble();
                idx++;
            }

            Result result = solveMCPS(n, K, conflicts, resources, capacities, windows, weights);
            mapper.writerWithDefaultPrettyPrinter().writeValue(new File(args[1]), result);

        } catch (Exception e) {
            e.printStackTrace();
            System.exit(1);
        }
    }
}
While Model B technically touches more profile elements, its grounding is entirely flawed. Model A understands that a corporate gift must be tailored to the recipient's professional status while subtly representing the giver's background, making it the only acceptable response.

clear; clc;

%% Load base case
mpc = case118;   % Replace with your case file if needed

%% Safe branches (1-based indexing)
safeBranches = [ ...
1 2 3 4 5 6 8 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 ...
28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 ...
51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 73 ...
74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 ...
97 98 99 100 101 102 103 104 105 106 107 108 109 110 111 112 114 115 ...
116 117 118 119 120 121 122 123 124 125 126 127 128 129 130 131 132 135 ...
136 137 138 139 140 141 142 143 144 145 146 147 148 149 150 151 152 153 ...
154 155 156 157 158 159 160 161 162 163 164 165 166 167 168 169 170 171 ...
172 173 174 175 178 179 180 181 182 185 186];

%% Parameters
numSafe = length(safeBranches);
totalPossible = nchoosek(numSafe, 2);
numOutages = 10585;  % Change this to control how many outage cases are simulated (max=10585)
if numOutages > totalPossible
    error('numOutages exceeds total number of safe N-2 combinations.');
end

results_all = cell(numOutages, 4);  % [branch1, branch2, success, MW lost or 'BLACK_OUT']

%% Run base power flow
fprintf('Running base case power flow...\n');
results = runpf(mpc);
if ~results.success
    error('Base case power flow did not converge!');
end
base_load = sum(mpc.bus(:, 3));
fprintf('Base case converged.\n\n');

%% Generate random unique N-2 combinations from safe branches
rng('shuffle');
combos = nchoosek(safeBranches, 2);
randIdx = randperm(size(combos, 1), numOutages);
selectedCombos = combos(randIdx, :);

%% Simulate
fprintf('Running %d safe N-2 contingencies...\n', numOutages);
for i = 1:numOutages
    outage_pair = selectedCombos(i, :);
    mpc_out = mpc;
    mpc_out.branch(outage_pair, :) = [];

    try
        results_out = runpf(mpc_out);
    catch
        results_out.success = 0;
    end

    if results_out.success
        new_load = sum(results_out.bus(:,3));
        MW_lost = base_load - new_load;
        if MW_lost < 0
            MW_lost = 0;
        end
        success_flag = 1;
    else
        MW_lost = 'BLACK_OUT';
        success_flag = 0;
    end

    results_all{i,1} = outage_pair(1);
    results_all{i,2} = outage_pair(2);
    results_all{i,3} = success_flag;
    results_all{i,4} = MW_lost;

    fprintf('Outage %d: Branches [%d %d], Result: %s, Load lost: %s MW\n', ...
        i, outage_pair(1), outage_pair(2), ternary(success_flag, 'OK', 'BLACK_OUT'), ...
        ternary(success_flag, num2str(MW_lost, '%.2f'), 'N/A'));
end

%% Save results to CSV
MW_lost_col = cell(numOutages,1);
for k = 1:numOutages
    val = results_all{k,4};
    if isnumeric(val)
        MW_lost_col{k} = num2str(val,'%.2f');
    else
        MW_lost_col{k} = val;
    end
end

dataTable = table([results_all{:,1}]', [results_all{:,2}]', [results_all{:,3}]', MW_lost_col, ...
    'VariableNames', {'Branch1','Branch2','Success','MW_Lost'});

writetable(dataTable, 'N2_safe_outage_results.csv');

%% Analyze blackouts - count how often each branch caused blackout
branchCount = containers.Map('KeyType','double','ValueType','double');
for k = 1:numOutages
    if ischar(results_all{k,4}) && strcmp(results_all{k,4}, 'BLACK_OUT')
        b1 = results_all{k,1};
        b2 = results_all{k,2};
        branchCount(b1) = getOrDefault(branchCount, b1, 0) + 1;
        branchCount(b2) = getOrDefault(branchCount, b2, 0) + 1;
    end
end

branches = cell2mat(keys(branchCount))';
counts = cell2mat(values(branchCount))';

branchTable = table(branches, counts, 'VariableNames', {'Branch','BlackoutCount'});
branchTable = sortrows(branchTable, 'BlackoutCount', 'descend');

writetable(branchTable, 'Branch_blackout_summary.csv');

fprintf('\nFiles saved:\n- N2_safe_outage_results.csv\n- Branch_blackout_summary.csv\n');

%% Helper functions
function out = ternary(cond, valTrue, valFalse)
    if cond
        out = valTrue;
    else
        out = valFalse;
    end
end

function val = getOrDefault(map, key, defaultVal)
    if isKey(map, key)
        val = map(key);
    else
        val = defaultVal;
    end
end
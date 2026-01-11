# NFL
*Analyzing player movement when the ball is in the air*



1. Understanding the Dataset
The dataset used in this project is a large-scale, frame-by-frame player-tracking table derived from NFL game film. Each row represents a single player at a specific frame of a specific play, captured at 10 frames per second, with positional coordinates (x, y).
The data spans across 18 weeks of the game, where every week is divided into input (data before the ball is thrown) and output (data after the ball is thrown) files. The output files are restricted to data pertaining exclusively to the players of interest.


2. Definitions and Assumptions
Definitions
Player to Predict: Defensive players explicitly marked for analysis, representing the relevant defensive coverage assignment for a given play.

Total frames: The frames utilized to capture the entire play.

Optimal Path: The shortest Euclidean path from the player's position at ball release to the ball's final landing location. This represents the theoretically perfect pursuit trajectory.

Ball Landing Coordinates: The final position of the ball catch/incompletion/interception (ball_land_x, ball_land_y), representing the target point for pursuit evaluation.

Path Timing Deviation: Longitudinal deviation measures how early or late a player is relative to the optimal schedule of arrival. If the player is ahead of schedule it is on the positive side of the graph, If a player is behind he is on behind of schedule. In case of zero, the player has a perfect arrival timing.

Directional Deviation: Latitudinal deviation measures how far left or right a player veers away from the optimal pursuit line. This is essentially the 2D cross product, giving a signed distance perpendicular to the optimal path. If a player drifts towards the left he is going in the positive direction of the graph and in negative direction if he is drifting towards the right side. In case of zero, the player is in a perfect directional alignment of the path he took.

Radial Excess: Radial excess evaluates whether the player stayed unnecessarily far from the ball compared to an optimal shrinking radius.

Pursuit Efficiency Index: The three components from the top are being normalized giving a metric that unifies timing, directional accuracy and distance to ball into a single interpretable score. If the player has a perfect pursuit, his PEI score will be 1.00 and in case of a highly inefficient movement, his PEI value would be 0.00.

Key Assumptions
Straight-line optimality: We assume the optimal pursuit path is a direct line from current position to ball landing location.

Uniform progress measurement: Linear interpolation provides a schedule for measuring expected progress along the optimal path, proportional to frame count.

Independence of frames: Each frame is evaluated independently for deviation metrics, though aggregation captures cumulative performance.

Completion Reward: If the targeted receiver catches the ball, the timing of the path was correct and therefore we forgive the path Mean Absolute Error(MAE). We also reduce the directional and radial MAE (0.3x).

Interception Bonus: If a defender intercepts the ball, they have executed the best pursuit path to make sure that the receiver does not get to the ball. Therefore we reward them the same way we do to a targeted receiver.

Tight Coverage Bonus: If a defender is within less than or equal to the targeted receiver across the final 10 frames of an incomplete pass, we apply a 50% reduction to their penalties and reward them with the good coverage even if they do not intercept the ball.


3. Merging the Data
To perform meaningful pursuit or efficiency analysis, the raw NFL tracking information must first be combined into a single unified dataset. Because no individual table contains all necessary information, a multi-stage merging process was required. The backbone of the merge is the player-tracking table, where each row represents a specific player in a specific frame of a play.

To enrich this movement data with football context, additional metadata tables are merged using shared identifiers: (game_id, play_id, nfl_id). Play-level metadata contributes information such as pass result (complete, incomplete, interception), pass length, route of the targeted receiver, and the ball’s landing coordinates. These attributes allow us to interpret tracking data not just as motion, but as an unfolding football event where timing, location, and roles matter.


4. Optimal Path (On Field)
The concept of the Optimal Path represents the ideal straight-line trajectory from a player’s position at the moment the ball is thrown to the exact location where the ball will land. Since the ball’s landing point is known for every play, this creates a fixed target that allows us to evaluate how efficiently each player moves towards or away from that destination. This gives both the direction and total distance that the player would need to cover in a perfect pursuit scenario. The Euclidean distance formula is used from the player's positions on x and y coordinate in the first output frame to x, y coordinate of ball landing point. By parameterizing this segment over time using a normalized progress variable alpha(t), which increases linearly from 0 at the throw frame to 1 at the catch or landing frame, we construct an “ideal” schedule of where the player should be along the path at each frame. This optimal trajectory serves as the baseline against which actual movement is compared. Any deviation in this case, whether falling behind, overrunning, or drifting sideways represents an inefficiency.


5. Path Dev (Longitudinal Deviation)
Path Deviation measures how far ahead or behind a player is compared to where they should be along the optimal straight-line path toward the ball. After establishing the optimal path length L and the unit direction vector u, each player’s actual displacement from their throw-frame starting point is projected onto this direction using the dot product:

along(t) = r(t) * u,

where r(t) is the player’s displacement vector at frame t. In parallel, the optimal progress at each frame is given by a time-scaled schedule,

along_opt(t) = alpha(t) * L,

where alpha(t) increases linearly from 0 (at the throw frame) to 1 (at the ball landing). Longitudinal Deviation is then defined as:

path_dev(t) = along(t) - along_opt(t).

A positive deviation means the player is ahead of the optimal chase schedule, while a negative deviation means they are behind and not covering ground fast enough. By analyzing this value across the full ball-in-air window, we can quantify timing.


6. Directional Dev (Lateral Deviation)
Directional Deviation measures how far a player strays sideways from the optimal pursuit line toward the ball. While Path deviation captures timing, Directional deviation captures steering accuracy, how well the player stays aligned with the ideal angle of pursuit. After defining the optimal direction vector u from the player’s throw-frame position to the ball, each player’s displacement vector r(t) is decomposed into perpendicular and parallel components. The lateral component is extracted using the 2-D cross product:

Dir_dev(t) = u_x * r_y(t) - u_y * r_x(t),

which produces a signed value indicating the side of deviation. Positive values mean drifting left of the optimal path, while negative values indicate drifting right. The magnitude |Dir_dev(t)| reveals how far the player is off-angle at that instant. Averaging these magnitudes across all output (ball-in-air) frames provides a measure of directional consistency in how well the player maintains the correct pursuit angle without taking inefficient lateral steps.


7. Radial Excess (Excess Distance above Optimal Range)
Radial Excess measures how much farther a player is from the ball than they need to be at each moment under an ideal pursuit model. While Path and Directional deviations capture timing and directional accuracy, radial excess captures overall pursuit inefficiency by quantifying the extra distance a player maintains from the ball beyond what is required at that frame. To compute this, we compare the player’s actual distance to the ball:

d_actual(t) = sqrt{(x(t)-x_b)^2 + (y(t)-y_b)^2}

against the optimal radial schedule, which linearly decreases from the throw-frame distance L down to zero at the catch point:

d_opt(t) = (1 - alpha(t)) * L

where alpha(t) is the normalized progress toward the catch. Radial Excess is then computed as:
rad_excess(t) = max(0, d_actual(t) - d_opt(t)), ensuring that players are not penalized for getting closer to the ball than expected. Instead, only inefficiencies in frames where a player lagged too far behind the ideal racing lines are counted.


8. PEI (Player Efficiency Index)
The Pursuit Efficiency Index (PEI) is the final metric developed to translate the three deviation components which are Path Deviation, Directional Deviation, and radial excess into a single interpretable score that measures how efficiently a player pursued the ball. To combine these into one measure, each deviation is normalized by the total straight-line distance to the ball, ensuring that players with longer pursuit paths are not unfairly penalized. The normalized deviations are averaged and inverted so that PEI scores range from 0 to 1, where 1 represents perfect, optimal pursuit. This approach preserves all three types of inefficiency, gives equal mathematical weight to angle, timing, and path distance, and produces a stable, comparable metric across players, positions, and plays.


9. Average PEI Over Season
After computing the Pursuit Efficiency Index (PEI) for every player on every targeted play, the next step was to evaluate the average of each player over the entire season. Because individual plays can be highly variable, single-play PEI values do not fully represent a player’s overall pursuit ability.

[Dashboard - Average PEI for 2023 Season by Position and Route]
https://lookerstudio.google.com/reporting/c22eb038-8776-4402-a5c4-22cbd19d4f5c/page/mQ8hF


10. Conclusion
The Pursuit Efficiency Index (PEI) integrates timing, angle discipline, and overall pursuit quality components into a single, interpretable metric. Beyond play-level evaluation, the seasonal aggregation of PEI highlights long-term trends in player performance, enabling meaningful comparisons across games, roles, and positions.

The results demonstrate that PEI provides insight not captured by traditional stats like tackles or pass breakups alone. Players with strong technique and fast ball landing point recognition consistently exhibit high PEI values, while those who take indirect or delayed pursuit routes show measurable inefficiencies. Incorporating contextual bonuses such as rewarding successful catches, interceptions, or tight coverage, ensures that the metric aligns with real football outcomes.

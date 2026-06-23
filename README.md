damn

#include <iostream>
#include <vector>

using namespace std;

void solve() {
    int n;
    cin >> n;

    long long answer = 0;
    int best;

    for (int i = 0; i < n; i++) {
        int value;
        cin >> value;

        if (i == 0) {
            best = value;
        } else if (value < best) {
            best = value;
        }

        answer += best;
    }

    cout << answer << '\n';
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int T;
    cin >> T;

    while (T--) {
        solve();
    }

    return 0;
}

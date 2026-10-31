// 5. CÂY BIỂU THỨC (CÂY NHỊ PHÂN) VÀ CÁC PHÉP DUYỆT
// ---------------------------------------------------------------------
struct TNode {
    Token t; TNode *l = nullptr, *r = nullptr;
    explicit TNode(const Token& tk) : t(tk) {}
};
void freeTree(TNode* n) { if (!n) return; freeTree(n->l); freeTree(n->r); delete n; }

TNode* buildTree(const vector<Token>& pf) {
    Stack<TNode*> st;
    auto fail = [&](const string& msg) {
        while (!st.empty()) { freeTree(st.top()); st.pop(); }
        throw runtime_error(msg);
    };
    for (const Token& t : pf) {
        TNode* n = new TNode(t);
        if (t.type == TokType::NUM || t.type == TokType::VAR) { st.push(n); continue; }
        bool unary = (t.text == "~") || t.type == TokType::FUNC;
        if (st.size() < (unary ? 1u : 2u)) { delete n; fail("Thieu toan hang"); }
        if (!unary) { n->r = st.top(); st.pop(); }
        n->l = st.top(); st.pop();
        st.push(n);
    }
    if (st.size() != 1) fail("Bieu thuc khong hop le");
    return st.top();
}

int nodePrec(const TNode* n) {
    return (n->t.type == TokType::OP) ? prec(n->t.text) : 5;   // lá, hàm: ưu tiên cao nhất
}
bool needParens(int childPrec, const string& parentOp, bool isRight) {
    if (parentOp.empty()) return false;
    int pp = prec(parentOp);
    if (childPrec > pp) return false;
    if (childPrec < pp) return true;
    return isRight ? !rightAssoc(parentOp) : rightAssoc(parentOp);
}

// Cây -> Trung tố (chỉ thêm ngoặc khi cần thiết)
string toInfix(const TNode* n, const string& parentOp = "", bool isRight = false) {
    string s;
    if (n->t.type == TokType::NUM || n->t.type == TokType::VAR) return n->t.text;
    if (n->t.type == TokType::FUNC) return n->t.text + "(" + toInfix(n->l) + ")";
    if (n->t.text == "~") s = "-" + toInfix(n->l, "~", false);
    else s = toInfix(n->l, n->t.text, false) + " " + n->t.text + " " + toInfix(n->r, n->t.text, true);
    return needParens(nodePrec(n), parentOp, isRight) ? "(" + s + ")" : s;
}
// Duyệt NLR -> Tiền tố
void toPrefix(const TNode* n, vector<string>& out) {
    if (!n) return;
    out.push_back(showName(n->t));
    toPrefix(n->l, out); toPrefix(n->r, out);
}
// Cây -> LaTeX
string toLatex(const TNode* n, const string& parentOp = "", bool isRight = false) {
    const Token& t = n->t;
    if (t.type == TokType::NUM) return t.text;
    if (t.type == TokType::VAR) return t.text == "pi" ? "\\pi" : t.text;
    if (t.type == TokType::FUNC) {
        string in = toLatex(n->l);
        if (t.text == "sqrt") return "\\sqrt{" + in + "}";
        if (t.text == "abs")  return "\\left|" + in + "\\right|";
        return "\\" + (t.text == "log" ? string("log") : t.text) + "\\left(" + in + "\\right)";
    }
    string s;
    if (t.text == "~") s = "-" + toLatex(n->l, "~", false);
    else if (t.text == "/") s = "\\frac{" + toLatex(n->l) + "}{" + toLatex(n->r) + "}";
    else if (t.text == "^") s = toLatex(n->l, "^", false) + "^{" + toLatex(n->r) + "}";
    else {
        string op = t.text == "*" ? " \\cdot " : (t.text == "%" ? " \\bmod " : " " + t.text + " ");
        s = toLatex(n->l, t.text, false) + op + toLatex(n->r, t.text, true);
    }
    int cp = nodePrec(n);
    if (t.text == "/" && !(parentOp == "^" && !isRight)) cp = 5;   // \frac tự đóng gói
    return needParens(cp, parentOp, isRight) ? "(" + s + ")" : s;
}
// Vẽ cây xoay ngang: gốc bên trái, con phải ở trên, con trái ở dưới
void printTree(const TNode* n, int depth = 0) {
    if (!n) return;
    printTree(n->r, depth + 1);
    cout << string(depth * 6, ' ') << "[" << showName(n->t) << "]\n";
    printTree(n->l, depth + 1);
}
int treeHeight(const TNode* n) { return n ? 1 + max(treeHeight(n->l), treeHeight(n->r)) : 0; }

# UIKit 实现模式

## 技术栈决策树

| 判断条件 | 推荐方案 | 说明 |
|---------|---------|------|
| 项目最低支持 iOS 13-15 | UIKit 为主 | SwiftUI 在低版本上功能受限 |
| 项目最低支持 iOS 16+ | SwiftUI 为主，UIKit 辅助 | 大部分场景 SwiftUI 可覆盖 |
| 项目最低支持 iOS 17+ | SwiftUI 优先 | @Observable 等现代 API 完备 |
| 已有大量 UIKit 代码的模块 | 继续用 UIKit | 保持一致性，避免混合架构 |
| 需要精细控制动画/手势/渲染 | UIKit | UIKit 对底层控制更强 |
| 快速搭建原型/简单页面 | SwiftUI | 声明式 UI 效率高 |
| 新模块，团队熟悉 SwiftUI | SwiftUI + 必要时 UIKit 补充 | 长期趋势 |

**原则：** 以项目现有技术栈为准。不要在 UIKit 项目中未经讨论擅自引入 SwiftUI，反之亦然。

---

## UIKit 视图构建模式

### Auto Layout 最佳实践

```swift
// 推荐：使用 NSLayoutConstraint.activate 批量激活约束
private func setupLayout() {
    // 关闭 autoresizing mask 转换
    [titleLabel, subtitleLabel, iconView].forEach {
        $0.translatesAutoresizingMaskIntoConstraints = false
        contentView.addSubview($0)
    }

    NSLayoutConstraint.activate([
        iconView.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: 16),
        iconView.centerYAnchor.constraint(equalTo: contentView.centerYAnchor),
        iconView.widthAnchor.constraint(equalToConstant: 40),
        iconView.heightAnchor.constraint(equalToConstant: 40),

        titleLabel.leadingAnchor.constraint(equalTo: iconView.trailingAnchor, constant: 12),
        titleLabel.trailingAnchor.constraint(lessThanOrEqualTo: contentView.trailingAnchor, constant: -16),
        titleLabel.topAnchor.constraint(equalTo: contentView.topAnchor, constant: 12),

        subtitleLabel.leadingAnchor.constraint(equalTo: titleLabel.leadingAnchor),
        subtitleLabel.trailingAnchor.constraint(equalTo: titleLabel.trailingAnchor),
        subtitleLabel.topAnchor.constraint(equalTo: titleLabel.bottomAnchor, constant: 4),
        subtitleLabel.bottomAnchor.constraint(equalTo: contentView.bottomAnchor, constant: -12),
    ])
}
```

**常见陷阱：**
- 忘记设置 `translatesAutoresizingMaskIntoConstraints = false`
- 约束冲突：检查是否有多余的 `intrinsicContentSize` 或优先级冲突
- Self-sizing Cell 需要确保垂直方向约束链完整（从 top 到 bottom）

### UIStackView 简化布局

```swift
// 推荐：用 UIStackView 减少约束代码
private lazy var contentStack: UIStackView = {
    let stack = UIStackView(arrangedSubviews: [titleLabel, subtitleLabel])
    stack.axis = .vertical
    stack.spacing = 4
    stack.alignment = .leading
    return stack
}()
```

### DiffableDataSource（iOS 13+）

```swift
// 定义 Section 和 Item
enum Section: Hashable {
    case main
    case featured
}

struct Item: Hashable {
    let id: String
    let title: String
}

// 配置数据源
private lazy var dataSource: UITableViewDiffableDataSource<Section, Item> = {
    UITableViewDiffableDataSource(tableView: tableView) { tableView, indexPath, item in
        let cell = tableView.dequeueReusableCell(withIdentifier: "Cell", for: indexPath)
        cell.textLabel?.text = item.title
        return cell
    }
}()

// 更新数据
func updateList(with items: [Item]) {
    var snapshot = NSDiffableDataSourceSnapshot<Section, Item>()
    snapshot.appendSections([.main])
    snapshot.appendItems(items, toSection: .main)
    dataSource.apply(snapshot, animatingDifferences: true)
}
```

### Compositional Layout（iOS 13+）

```swift
// 创建现代 Collection View 布局
func createLayout() -> UICollectionViewCompositionalLayout {
    // 每个 item 占满一行
    let itemSize = NSCollectionLayoutSize(
        widthDimension: .fractionalWidth(1.0),
        heightDimension: .estimated(80)
    )
    let item = NSCollectionLayoutItem(layoutSize: itemSize)

    let groupSize = NSCollectionLayoutSize(
        widthDimension: .fractionalWidth(1.0),
        heightDimension: .estimated(80)
    )
    let group = NSCollectionLayoutGroup.vertical(layoutSize: groupSize, subitems: [item])

    let section = NSCollectionLayoutSection(group: group)
    section.interGroupSpacing = 8
    section.contentInsets = NSDirectionalEdgeInsets(top: 16, leading: 16, bottom: 16, trailing: 16)

    return UICollectionViewCompositionalLayout(section: section)
}

// 横向滚动卡片布局
func createHorizontalCardSection() -> NSCollectionLayoutSection {
    let itemSize = NSCollectionLayoutSize(
        widthDimension: .fractionalWidth(1.0),
        heightDimension: .fractionalHeight(1.0)
    )
    let item = NSCollectionLayoutItem(layoutSize: itemSize)

    let groupSize = NSCollectionLayoutSize(
        widthDimension: .fractionalWidth(0.8),
        heightDimension: .absolute(200)
    )
    let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item])

    let section = NSCollectionLayoutSection(group: group)
    section.orthogonalScrollingBehavior = .groupPagingCentered
    section.interGroupSpacing = 12
    return section
}
```

---

## UIKit + MVVM 模板

### ViewController + ViewModel 绑定

```swift
// MARK: - ViewModel
@MainActor
@Observable
final class ProfileViewModel {
    enum ViewState {
        case idle, loading
        case loaded(Profile)
        case error(String)
    }

    private(set) var state: ViewState = .idle
    private let service: ProfileServiceProtocol

    init(service: ProfileServiceProtocol) {
        self.service = service
    }

    func loadProfile() async {
        state = .loading
        do {
            let profile = try await service.fetchProfile()
            state = .loaded(profile)
        } catch {
            state = .error(error.localizedDescription)
        }
    }
}

// MARK: - ViewController
final class ProfileViewController: UIViewController {
    private let viewModel: ProfileViewModel

    init(viewModel: ProfileViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) { fatalError("init(coder:) has not been implemented") }

    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        observeViewModel()
        Task { await viewModel.loadProfile() }
    }

    // 使用 withObservationTracking 绑定状态（iOS 17+）
    private func observeViewModel() {
        withObservationTracking {
            _ = viewModel.state
        } onChange: { [weak self] in
            Task { @MainActor in
                self?.updateUI()
                self?.observeViewModel() // 重新注册观察
            }
        }
    }

    private func updateUI() {
        switch viewModel.state {
        case .idle: break
        case .loading: showLoading()
        case .loaded(let profile): showProfile(profile)
        case .error(let message): showError(message)
        }
    }
}
```

**iOS 17 以下的绑定方式：** 使用 Combine 的 `@Published` + `sink`，或者 closure/delegate 回调。

---

## UIKit 导航模式：Coordinator Pattern

```swift
// MARK: - Coordinator 协议
protocol Coordinator: AnyObject {
    var childCoordinators: [Coordinator] { get set }
    var navigationController: UINavigationController { get }
    func start()
}

// MARK: - 具体 Coordinator
final class ProfileCoordinator: Coordinator {
    var childCoordinators: [Coordinator] = []
    let navigationController: UINavigationController

    init(navigationController: UINavigationController) {
        self.navigationController = navigationController
    }

    func start() {
        let viewModel = ProfileViewModel(service: ProfileService())
        let viewController = ProfileViewController(viewModel: viewModel)
        viewController.coordinator = self
        navigationController.pushViewController(viewController, animated: true)
    }

    func showEditProfile(_ profile: Profile) {
        let editVC = EditProfileViewController(profile: profile)
        navigationController.pushViewController(editVC, animated: true)
    }

    func showSettings() {
        let settingsCoordinator = SettingsCoordinator(navigationController: navigationController)
        childCoordinators.append(settingsCoordinator)
        settingsCoordinator.start()
    }
}
```

**适用场景：** 页面导航逻辑复杂、需要解耦 VC 间依赖、多入口跳转同一页面。

---

## UIKit ↔ SwiftUI 混编

### 在 UIKit 中嵌入 SwiftUI（UIHostingController）

```swift
// 将 SwiftUI 视图嵌入 UIKit 页面
let swiftUIView = ProfileCardView(profile: profile)
let hostingController = UIHostingController(rootView: swiftUIView)

// 方式一：作为子 VC
addChild(hostingController)
view.addSubview(hostingController.view)
hostingController.view.translatesAutoresizingMaskIntoConstraints = false
NSLayoutConstraint.activate([
    hostingController.view.topAnchor.constraint(equalTo: headerView.bottomAnchor),
    hostingController.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
    hostingController.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
    hostingController.view.heightAnchor.constraint(equalToConstant: 200),
])
hostingController.didMove(toParent: self)

// 方式二：直接 push
navigationController?.pushViewController(hostingController, animated: true)
```

### 在 SwiftUI 中嵌入 UIKit（UIViewRepresentable）

```swift
struct MapViewWrapper: UIViewRepresentable {
    @Binding var region: MKCoordinateRegion

    func makeUIView(context: Context) -> MKMapView {
        let mapView = MKMapView()
        mapView.delegate = context.coordinator
        return mapView
    }

    func updateUIView(_ mapView: MKMapView, context: Context) {
        mapView.setRegion(region, animated: true)
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, MKMapViewDelegate {
        let parent: MapViewWrapper
        init(_ parent: MapViewWrapper) { self.parent = parent }
    }
}
```

### 在 SwiftUI 中嵌入 UIViewController（UIViewControllerRepresentable）

```swift
struct ImagePickerView: UIViewControllerRepresentable {
    @Binding var selectedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: ImagePickerView
        init(_ parent: ImagePickerView) { self.parent = parent }

        func imagePickerController(_ picker: UIImagePickerController,
                                   didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]) {
            parent.selectedImage = info[.originalImage] as? UIImage
            parent.dismiss()
        }
    }
}
```

### 混编注意事项

| 注意点 | 说明 |
|-------|------|
| 数据传递 | UIKit → SwiftUI 通过 `rootView` 属性或 `@Binding`；SwiftUI → UIKit 通过 Coordinator 回调 |
| 生命周期 | `UIHostingController` 遵循 UIKit 生命周期，`viewDidAppear` 等正常工作 |
| 导航栏 | 混编时注意 SwiftUI 的 `navigationTitle` 和 UIKit 的 `title` 可能冲突 |
| 性能 | 频繁创建 `UIHostingController` 有开销，复用而非重建 |
| 主题/样式 | 确保 UIKit 和 SwiftUI 部分的颜色、字体风格一致 |
